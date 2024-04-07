## 一、[基础架构](https://blog.csdn.net/weixin_48154829/category_12373957.html?spm=1001.2014.3001.5482)

###  1.1 概述

主要分为 计算层（TiDB server）、存储层（TiKV、TiFlash）、管理节点（PD）。

![](https://img-blog.csdnimg.cn/19026c28a6cf477f881a9aaf1c85d1d4.png)

<br/>

### 1.2 计算层

- 处理客户端建联
- mysql协议解析
- gc
- sql执行

### 1.3.1 存储层-TiKV

- rocksdb 落盘
-  mvcc
- region group内raft协议保证多副本数据一致性
- 行存适合OLTP，是TiKV承载的功能；

### 1.3.2 存储层-TiFlash

- TiFlash存储的数据和 TiKV是一样的 ， 是TiKV的列存储版本
- 列存适合OLAP ，是TiFlash承载的功能，比如暴力扫描 ，分析数据，生成报表等

###  1.4 管理层

- 生成TSO
- 负载均衡
- 元数据管理

## [Raft](https://blog.csdn.net/weixin_48154829/article/details/131766046)

![](https://pic2.zhimg.com/v2-6199c4b5150737aa28f158a06fa8bb09_r.jpg)

TiKV 使用 Raft 一致性算法来保证数据的安全，默认提供的是三个副本支持，**这三个副本形成了一个 Raft Group**。

写：

- 当 Client 需要写入某个数据的时候，Client 会将操作发送给 Raft Leader，这个在 TiKV 里面我们叫做 **Propose**。
- Leader 会将操作编码成一个 **entry**，写入到自己的 Raft Log 里面，这个我们叫做 **Append**。
- Leader 也会通过 Raft 算法将 entry 复制到其他的 Follower 上面，这个我们叫做 **Replicate**。
- Follower 收到这个 entry 之后也会同样进行 Append 操作，顺带告诉 Leader Append 成功。当 Leader 发现这个 entry 已经被大多数节点 Append，就认为这个 entry 已经是 Committed 的了，然后就可以将 entry 里面的操作解码出来，执行并且应用到状态机里面，这个我们叫做 **Apply**。

读：

在 TiKV 里面，我们提供了 Lease Read，对于 Read 请求，会直接发给 Leader，如果 Leader 确定自己的 lease 没有过期，那么就会直接提供 Read 服务，这样就不用走一次 Raft 了。如果 Leader 发现 lease 过期了，就会强制走一次 Raft 进行续租，然后再提供 Read 服务。

<br/>

## Multi Raft

![](https://pic4.zhimg.com/v2-cdfe9eadb97a3d203cc670f488af3cff_r.jpg)

因为一个 Raft Group 处理的数据量有限，所以我们会将数据切分成多个 Raft Group，我们叫做 Region。

切分的方式是按照 range 进行切分，也就是我们会将数据的 key 按照字节序进行排序，也就是一个无限的 sorted map，然后将其切分成一段一段（连续）的 key range，每个 key range 当成一个 Region。

两个相邻的 Region 之间不允许出现空洞，也就是前面一个 Region 的 end key 就是后一个 Region 的 start key。Region 的 range 使用的是**前闭后开**的模式 [start, end)，对于 key start 来说，它就属于这个 Region，但对于 end 来说，它其实属于下一个 Region。

TiKV 的 Region 会有最大 size 的限制，当超过这个阈值之后，就会**分裂**成两个 Region，譬如 [a, b) -> [a, ab) + [ab, b)，当然，如果 Region 里面没有数据，或者只有很少的数据，也会跟相邻的 Region 进行**合并**，变成一个更大的 Region，譬如 [a, ab) + [ab, b) -> [a, b)。

## Percolator 

首先，Percolator 需要一个服务 timestamp oracle (TSO) 来分配全局的 timestamp，这个 timestamp 是按照时间单调递增的，而且全局唯一。

任何事务在开始的时候会先拿一个 **start timestamp (startTS)**，然后在事务提交的时候会拿一个 **commit timestamp (commitTS)**。

Percolator 提供**三个 column family (CF)，Lock，Data 和 Write**。

当写入一个 key-value 的时候，会将这个 key 的 lock 放到** Lock CF** 里面，会将实际的 value 放到** Data CF **里面，如果这次写入 commit 成功，则会将对应的 commit 信息放到入** Write CF **里面。

Key 在 Data CF 和 Write CF 里面存放的时候，会把对应的时间戳给加到 Key 的后面。在 Data CF 里面，添加的是 startTS，而在 Write CF 里面，则是 commitCF。

假设我们需要写入 a = 1，首先从 TSO 上面拿到一个 startTS，譬如 10，然后我们进入 Percolator 的 PreWrite 阶段，在 Lock 和 Data CF 上面写入数据，如下：

> Lock CF: W a = lock
Data CF: W a_10 = value

**W 表示 Write，R 表示 Read， D 表示 Delete，S 表示 Seek**

当 PreWrite 成功之后，就会进入 Commit 阶段，会从 TSO 拿一个 commitTS，譬如 11，然后写入：

> Lock CF: D a
Write CF: W a_11 = 10

当 Commit 成功之后，对于一个 key-value 来说，它就会在 Data CF 和 Write CF 里面都有记录，在 Data CF 里面会记录实际的数据， Write CF 里面则会记录对应的 startTS。

<br/>

当我们要**读取数据**的时候，也会先从 TSO 拿到一个 startTS，譬如 12，然后进行读：

> Lock CF: R a
Write CF: S a_12 -> a_11 = 10
Data CF: R a_10

在 Read 流程里面，首先我们看 Lock CF 里面是否有 lock，如果有，那么读取就失败了。

如果没有，我们就会在 Write CF 里面 seek 最新的一个提交版本，这里我们会找到 11，然后拿到对应的 startTS，这里就是 10，然后将 key 和 startTS 组合在 Data CF 里面读取对应的数据。
上面只是简单的介绍了下 Percolator 的读写流程，实际会比这个复杂的多。

## RocksDB

TiKV 会将数据存储到 RocksDB，RocksDB 是一个 key-value 存储系统，所以对于 TiKV 来说，任何的数据都最终会转换成一个或者多个 key-value 存放到 RocksDB 里面。

每个 TiKV 包含两个 RocksDB 实例，一个用于存储 Raft Log，我们后面称为 **Raft RocksDB**，而另一个则是存放用户实际的数据，我们称为** KV RocksDB**。

一个 TiKV 会有多个 Regions，我们在 Raft RocksDB 里面会使用 Region 的 ID 作为 key 的前缀，然后再带上 Raft Log ID 来唯一标识一条 Raft Log。譬如，假设现在有两个 Region，ID 分别为 1，2，那么 Raft Log 在 RocksDB 里面类似如下存放：

> 1_1 -> Log {a = 1}
1_2 -> Log {a = 2}
…
1_N -> Log {a = N}
2_1 -> Log {b = 2}
2_2 -> Log {b = 3}
…
2_N -> Log {b = N}

因为我们是按照 range 对 key 进行的切分，那么在 KV RocksDB 里面，我们直接使用 key 来进行保存，类似如下：

> a -> N
b -> N

RocksDB 支持 Column Family，所以能直接跟 Percolator 里面的 CF 对应，在 TiKV 里面，我们在 RocksDB 使用 Default CF 直接对应 Percolator 的 Data CF，另外使用了相同名字的 Lock 和 Write。

<br/>

## API

TiKV 提供两套 API，一套叫做 **RawKV**，另一套叫做 **TxnKV**。

TxnKV 对应的就是上面提到的 Percolator，而 RawKV 则不会对事务做任何保证。

#### RawKV-write

![](https://pic4.zhimg.com/v2-1fd4a870d321c5da5c89ad8b1e17ccd3_r.jpg)

#### RawKV-read

![](https://pic2.zhimg.com/v2-2f6356ecb30b884ce25ceec89af1c519_r.jpg)

####  TxnKV-Write

![](https://pic1.zhimg.com/v2-8b72f207ab9b77f534c6393acb27b488_r.jpg)

#### TxnKV-read

![](https://pic2.zhimg.com/v2-e3239f73c2c07d91376dd4fd8a766bc1_r.jpg)

<br/>

## SQL Key Mapping

```sql
CREATE TABLE t1 {
    id BIGINT PRIMARY KEY,
    name VARCHAR(1024),
    age BIGINT,
    content BLOB,
    UNIQUE(name),
    INDEX(age),
}

```

在 TiDB 里面，任何一张表都有一个唯一的 ID，譬如这里是 11，任何的索引也有唯一的 ID，上面 name 就是 12，age 就是 13。我们使用前缀 t 和 i 来区分表里面的 data 和 index。对于上面表 t1 来说，假设现在它有两行数据，分别是 (1, “a”, 10, “hello”) 和 (2, “b”, 12, “world”)，在 TiKV 里面，每一行数据会有不同的 key-value 对应。如下：

t + Table ID + PK 来唯一表示一行数据，value 就是这行数据

i + Index ID + name 来表示，而 value 则是对应的 PK

对于普通的 Index 来说，不需要唯一性约束，所以我们使用 i + Index ID + age + PK，而 value 为空。

```sh
PK
t_11_1 -> (1, “a”, 10, “hello”)
t_11_2 -> (2, “b”, 12, “world”)

Unique Name
i_12_a -> 1
i_12_b -> 2

Index Age
i_13_10_1 -> nil
i_13_12_2 -> nil
```

<br/>

---

<br/>

https://zhuanlan.zhihu.com/p/46524530/?utm_id=0
