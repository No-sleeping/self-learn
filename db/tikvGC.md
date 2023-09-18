# 一、原理

GC作为TiDB Server的模块，主要的作用就是清理不再需要的旧数据。

在做数据 update 时，不会更新原来的数据，而是重新写一份新的数据，并且给每份数据都给定一个事务时间戳，这种就是 MVCC（多版本并发控制）机制，为了避免同一条记录的版本过多，TiDB设定了一个 tikv\_gc\_life\_time 来控制版本的保留时间(默认10min)，并且每经过 tikv\_gc\_run\_interval 这个时间间隔，就会触发GC的操作，清除历史版本的记录，下面详细聊下GC的流程。选举 GC Leader 的方式很简单，GC Worker 每分钟 Tick 时，如果发现没有 Leader 或 Leader 失效，就把自己写进去，成为 GC Leader。

## 1.0 概念

**Safepoint**

每次 GC 时，首先 TiDB 会计算一个称为 Safepoint 的时间戳，接下来 TiDB 会在保证 Safepoint 之后的快照全部拥有正确数据的前提下，删除更早的过期数据。

<br/>

## 1.1 流程

GC都是由**GC Leader**触发的，一个tidb server的cluster会选举出一个GC leader，具体是哪个节点可以查看mysql.tidb表，gc leader只是一个身份状态，**真正run GC的是该Leader节点的gc worker。**

- （1）Resolve Locks 阶段

TiDB 的事务是基于** Google Percolator** 模型实现的，事务的提交是一个两阶段提交的过程。第一阶段完成时，所有涉及的 key 都会上锁，其中一个锁会被选为 Primary，其余的锁 ( Secondary ) 则会存储一个指向 Primary 的指针；第二阶段会将 Primary 锁所在的 key 加上一个 Write 记录，并去除锁。

如果因为某些原因（如发生故障等），这些 Secondary 锁没有完成替换、残留了下来，那么也可以根据锁中的信息找到 Primary，并根据 Primary 是否提交来判断整个事务是否提交。但是，如果 Primary 的信息在 GC 中被删除了，而该事务又存在未成功提交的 Secondary 锁，那么就永远无法得知该锁是否可以提交。这样，数据的正确性就无法保证。

Resolve Locks 这一步的任务即对 Safepoint 之前的锁进行清理。**即如果一个锁对应的 Primary 已经提交，那么该锁也应该被提交**；反之，则应该回滚。而如果 Primary 仍然是上锁的状态（没有提交也没有回滚），则应当将该事务视为超时失败而回滚。

Resolve Locks 的执行方式是由 GC leader 对所有的 Region 发送请求扫描过期的锁，并对扫到的锁查询 Primary 的状态，再发送请求对其进行提交或回滚。

<br/>

- （2）Delete Ranges阶段

在执行 Drop/Truncate Table ，Drop Index 等操作时，会有大量连续的数据被删除。如果对每个 key 都进行删除操作、再对每个 key 进行 GC 的话，那么执行效率和空间回收速度都可能非常的低下。事实上，这种时候 TiDB 并不会对每个 key 进行删除操作，而是将这些待删除的区间及删除操作的时间戳记录下来。**Delete Ranges 会将这些时间戳在 Safepoint 之前的区间进行快速的物理删除，而普通 DML 的多版本不在这个阶段回收。**

Drop/Truncate Table ，Drop Index 会先把 Ranges 写进 TiDB 系统表（`mysql.gc_delete_range`），TiDB 的 GC worker 定期查看是否过了 Safepoint，然后拿出这些 Ranges，**并发**的给 TiKV 去删除 sst 文件，并发数和 concurrency 无关，而是直接发给各个 TiKV。删除是直接删除，不需要等 compact 。完成 Delete Ranges 后，会记录在 TiDB 系统表 `mysql.gc_delete_range_done`，表中的内容过 24 小时后会清除:

```mysql
mysql> select * from gc_delete_range_done;
+--------+------------+--------------------+--------------------+--------------------+
| job_id | element_id | start_key          | end_key            | ts                 |
+--------+------------+--------------------+--------------------+--------------------+
|    283 |        171 | 7480000000000000ab | 7480000000000000ac | 422048703668289538 |
|    283 |        172 | 7480000000000000ac | 7480000000000000ad | 422048703668289538 |
+--------+------------+--------------------+--------------------+--------------------+
2 rows in set (0.01 sec)
```

<br/>

- （3）Do GC阶段

这一步主要是针对 DML 操作产生的 key 的过期版本进行删除。为了保证 Safepoint 之后的任何时间戳都具有一致的快照，这一步删除 Safepoint 之前提交的数据，但是会对每个 key 保留 Safepoint 前的最后一次写入（除非最后一次写入是删除）。

**在进行这一步时，TiDB 只需将 Safepoint 发送给 PD，即可结束整轮 GC。TiKV 会每 1 分钟自行检测是否 Safepoint 发生了更新，然后会对当前节点上所有 Region Leader 进行 GC**。与此同时，GC Leader 可以继续触发下一轮 GC。

<br/>

----

https://docs.pingcap.com/zh/tidb/stable/garbage-collection-overview#gc-%E6%9C%BA%E5%88%B6%E7%AE%80%E4%BB%8B  （GC 机制简介）

https://tidb.net/blog/36c58d32#%E5%87%8C%E6%99%A8%E7%9A%84%E6%8A%A5%E8%AD%A6%E5%92%8C%E5%A4%84%E7%90%86  （TiCDC异常引发的GC不干活导致的Tikv硬盘使用问题）

https://tidb.net/blog/ed740c2c（TiDB GC 之原理浅析）
