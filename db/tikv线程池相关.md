# 一、总

tikv 的线程池主要由以下几部分组成：

- gRPC
- Scheduler
- UnifyReadPool
- Raftstore
- StoreWriter
- Apply
- RocksDB
- scheduled tasks 
- detection components

这篇文章主要介绍一些线程池，这些线程池一般cpu敏感，但是会影响到读写性能。

# 二、介绍

- The gRPC thread pool: it handles all network requests and forwards requests of different task types to different thread pools.（处理grpc的网络请求和并分配不同任务到不同线程池）
- The Scheduler thread pool: it detects write transaction conflicts, converts requests like the two-phase commit, pessimistic locking, and transaction rollbacks into key-value pair arrays, and then sends them to the Raftstore thread for Raft log replication.（主要探测事务写冲突，将两阶段提交、悲观锁、事务回滚转换成kv数组对，然后发给raftstore 和 raftlog）
- The Raftstore thread pool:
  - It processes all Raft messages and the proposal to add a new log.（处理所有的raft信息）
  - It writes Raft logs to the disk. If the value of store-io-pool-size is 0, the Raftstore thread writes the logs to the disk; if the value is not 0, the Raftstore thread sends the logs to the StoreWriter thread.（将raft日志写到磁盘。如果store-io-pool-size==0，就直接将log写到磁盘；否则就将log先发送给storeWriter）
  - When Raft logs in the majority of replicas are consistent, the Raftstore thread sends the logs to the Apply thread.（当大多数副本同步到了raftlog，同时会将这些log发给apply线程）
- The StoreWriter thread pool: it writes all Raft logs to the disk and returns the result to the Raftstore thread.（将raftlog 写到磁盘，并将结果返回给raftstore线程）
- The Apply thread pool: it receives the submitted log sent from the Raftstore thread pool, parses it as a key-value request, then writes it to RocksDB, calls the callback function to notify the gRPC thread pool that the write request is complete, and returns the result to the client.（收到Raftstore线程提交的log，然后写入到RockDB，通过回调函数通知gRPC：写请求完成，并且将结果返回给客户端）
- The RocksDB thread pool: it is a thread pool for RocksDB to compact and flush tasks. For RocksDB's architecture and Compact operation, refer to RocksDB: A Persistent Key-Value Store for Flash and RAM Storage.（这个线程主要用于rocksDB的compact和flush任务。）
- The UnifyReadPool thread pool: it is a combination of the Coprocessor thread pool and Storage Read Pool. All read requests such as kv get, kv batch get, raw kv get, and coprocessor are executed in this thread pool.（所有的读请求都会在这个线程池内）

<br/>

<br/>

---

https://docs.pingcap.com/tidb/stable/tune-tikv-thread-performance
