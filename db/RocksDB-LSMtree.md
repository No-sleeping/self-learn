## 0、基础概念

#### 写多读少 vs 读多写少

读多写少：mysql 中的 innodb 存储引擎，底层基于 B+ Tree 这一数据结构进行存储文件

写多读少：日志、历史记录，kv 型存储组件 rocksdb 及其底层采用到的 lsm tree（log structured merge tree）

#### 原地写 vs 追加写

![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gNcabibACv8wJ4zyR6Ub7IpItYB7ghszibibiaoSUuqXwNCRLF6oz6S8hPeg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### 原地写：

倘若要针对一组 kv 数据执行**写操作**，首先要找到 kv 老数据的所在位置，再在其基础之上执行进行更新. 这个过程涉及到**磁盘的随机 IO**，因此性能较差.

与之相对，在执行**读操作**时，可以根据 k 寻找到 kv 数据所在位置并直接拿到查询结果，因此读操作效率相对较高. 且具有着不错的空间利用率.

##### 追加写：

追加写类型的写操作中，无须区分本次写操作是**插入还是更新**，而是选择将 kv 对以追加的形式直接插入到文件的末尾位置. 因此不涉及磁盘的随机 IO，只需要执行顺序 IO 操作，在写流程中的执行性能相较于原地写而言有较大的提升. 会存在**空间浪费**的情况.

<br/>

<br/>

## 1、LSM tree

该数据结构主要解决**写多读少**的场景，故采用**顺序写**的方式。

### 1.1 顺序写问题及措施

|问题描述|应对措施|措施描述|
|--|--|--|
|数据冗余（空间浪费）：除了最后一笔记录之外，此前多笔老数据都属于无用的冗余数据|1、数据合并：异步启动一个负责压缩合并的线程，持续对重复的 kv 对数据进行合并，只保留最新记录，消除此前无用的老数据.（会产生对同一文件同写、压缩的互斥操作，）|![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gNOuribicwbgkN8ecwrefzMUp6HKrtsAcUXCDllfFGE7HApmY9yVVhib1nw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)|
||2、文件分块：将写的数据块和压缩的数据隔离，解决上述互斥问题。|![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gN4pXgicO2hoFv6GbLiaj3knrebDPLIl3b6xSSmHhUoibQ4Bywjkn6ontXw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)|
|读性能低：执行一次查询操作需要反向追溯，直到找到第一笔满足条件的 kv 数据记录才能返回. 因此，其最坏的时间复杂度是线性的 O(N)|1、数据有序存储：事先根据 k 进行数据排序，那么在后续的查询环节中，我们就无需承受线性遍历的代价|![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gNjEHmicQNGpCbeS8pzWfWAOA6D29BlG4SDtRl5S5JyOdAnhPUnQ7C4Dg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)|

<br/>

### 1.2 	内存+磁盘

先明确信息：

-  memtable 在内存中缓存
- memtable 本身基于 k 进行有序存储
- memtable 中的写操作统一采用就地写
- memtable 是用户写操作的唯一入口
- memtable 数据量达到阈值后溢写到磁盘，成为 disktable

重要的事情，我们再次强调一遍，内存中的 memtable 采用就地写操作，而非追加写. 因为数据是维护在内存中的，因此哪怕写操作需要承受随机 IO 的代价，也处在可以接受的范围以内.

此外，当 memtable 中的数据量达到阈值后，再一次性 flush 到磁盘中成为 disktable. 这个过程中实际上是以 table 为粒度，在磁盘中执行了我们所谓的“追加写”操作.

这样做还带来的一点好处是，由于所有的 disk table 都是由内部有序的 memtable 生成的，因此能做到 disktable 文件内部天然就是局部有序的.

<br/>

<br/>

## 2、总结

lsm tree 全称 Log Structure Merge Tree，其核心设定如下：

• 存储介质主要依赖磁盘（sstable），但上层也会借助内存的辅助（memtable）

• 内存（memtable）就地写，磁盘（sstable）顺序写

• 写入口为可读可写的 active memtable

• 达到阈值后 active memtable 转为只读的 readonly memtable

• memtable 保证有序，默认基于跳表实现

• 由于 sstable 来自 memtable，每个sstable 内部无冗余数据且有序

• 磁盘文件分层（level0~levelk），上层为近期写入的热数据，下层为较早写入的冷数据

• level(i+1) sstable 大小恒定为 level(i) 层 T 倍

•** level0 sstable 之间存在冗余数据**

• **level1~levelk 单层内无冗余数据且全局有序**

• 数据沿着 level 0 -> level k 的方向合并，自顶向下流动

![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gNlkAf4C6eURh8wic0ThTaiacNHI7Orp72lVnMqd8YLqibHs0NkibFC0Va2g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

<br/>

### 2.1  读写流程

#### 写流程

![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gNqIEwqwlhQEiaftnsWFZCfCSic8o0uP20T3zM3ZBCyI7O20aB6lFT7pHA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

• 先写入磁盘的预写日志WAL，以防止内存数据丢失

• 基于就地写模式，写入内存中的 active memtable

• active memtable 达到阈值后转为只读的 readonly memtable

• readonly memtable 会 flush 到磁盘，成为 level0 的 sstable

• level(i) 层数据容量达到后，会基于归并的方式合并到 level(i+1)，以此类推

<br/>

#### 读流程

![](https://mmbiz.qpic.cn/mmbiz_png/3ic3aBqT2ibZvlBWa9iaEhdicBxGLZZDq5gNqZkOwalnbYMNQPfo9UeHCoq5bWPibia9kGbN7KenCMN82I1F2BMwDHEw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

• 尝试读 active memtable

• 尝试读 readonly memtable

• 尝试读 level0，需要按照溢写顺序进行倒序，依次读 level0 中的每个 sstable（level0 sstable 间数据可能冗余）

• 根据全局的索引文件，依次读 level1~levelk，每个 level 最多只需要读一个 sstable

• 读一个 sstable 时，借助内部的 bloom filter 和索引，加速查询流程

<br/>

<br/>

## 3、RocksDB Compaction 策略 

### 3.1  策略介绍

### 3.2 RocksDB 使用情况

---

<br/>

参考：

[ 初探 rocksdb 之 lsm tree](https://mp.weixin.qq.com/s/kqpBZ2aCC0CGvvL2Lm6mzA)  --- [视频讲解](https://www.bilibili.com/video/BV11u411P7GP/?p=10&spm_id_from=pageDriver)

[Rocksdb LSM Tree Compaction策略](https://blog.csdn.net/weixin_43778179/article/details/134027896)
