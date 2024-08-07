## tikv 架构

![](https://download.pingcap.com/images/docs/tikv-rocksdb.png)

作为TiKV 存储引擎，RocksDB主要存储raft log（raftdb） 和用户数据（kvdb）。

这两部分分别存储在rocksDB的两个实例里。

以下是kvdb 的四个CF：

- raft CF：
  - 每个region的元数据。所占空间不大。
- lock CF：
  - 存悲观事务的悲观锁和分布式事物的预写锁。当事务提交之后，lock CF里面相应数据会被删除。因此lockCF的数据大小通常很小（小于1GB）。如果该数据增长较快，那意味着有大量的数据等待着被提交，系统可能出现问题。
- write CF：
  - 存用户真正写的数据，和mvcc的元数据。如果用户写入数据小于225B，会存在writeCF内，否则是存在defaultCF内。在 TiDB 中，二级索引只占用writeC 的空间，因为非唯一索引存储的值为空，唯一索引存储的值为主键索引。
- default CF：存储长度超过 255 字节的数据。

## RocksDB memory usage

为了提升读性能，减少读对磁盘的操作，rocksdb 将存在磁盘上的文件基于实际大小划分（64KB）到blocks。当读一个block 的时候，会先校验数据是否已经存在于内存的blockcache。如果存在，就直接从mem中读取，不访问磁盘。

blockcache会和根据LRU算法释放最近使用的数据。默认情况下tikv 会将45%的men 分配给blockcache。用户可以通过修改`storage.block-cache.capacity`来调整，该值不建议超过60%。

写入rocksdb的数据会先写入memtable，当memtable的大小超过128MB， 会生成一个新的memtable。在TiKV中总共有两个rocksdb实例，总共4个CF，每个memtable的每个cf的容量上线是128MB。同时最大可以存在5个memtable，否则就会阻塞写入。这部分mem 最多会占用2.5GB（`4*5*128MB`），这个配置不建议改。

## RocksDB background threads and compaction

在rocksdb中，memtable -- sst 文件的转换、不同层sst文件的合并 ，这些都是在后台线程池中进行的。

默认的线程池数是：8。当机器的cpu核数小于等于8时，线程池的默认大小是cpu个数减一。

总的来说，用户不需要更改这些配置。如果用户在同一台机器上部署多个tikv 实例，或者机器当前的读负载相对高，写负载相对低，可以通过调整`rocksdb/max-background-jobs`，调整到3、4 可能是一个合适的配置。

## WriteStall

rocksdb的 L0 和其他层不太一样，整体上是有序的。key在sst中分布会有重叠，一个请求来的时候，会查询每一个sst文件。为了不影响查询性能，当有太多L0文件的时候，writestall就是触发阻塞写。

 如果遇到写时延突然陡增，你应该首先检查`WriteStall Reason`监控指标。

如果是L0 文件太多导致的，你可以把以下参数调整到64

```
rocksdb.defaultcf.level0-slowdown-writes-trigger
rocksdb.writecf.level0-slowdown-writes-trigger
rocksdb.lockcf.level0-slowdown-writes-trigger
rocksdb.defaultcf.level0-stop-writes-trigger
rocksdb.writecf.level0-stop-writes-trigger
rocksdb.lockcf.level0-stop-writes-trigger
```

<br/>

---

https://docs.pingcap.com/tidb/stable/rocksdb-overview#rocksdb-background-threads-and-compaction
