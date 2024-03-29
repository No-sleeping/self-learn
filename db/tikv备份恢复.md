

## 0、概述

基于 Raft 协议和合理的部署拓扑规划，TiDB 实现了集群的高可用，当集群中少数节点挂掉时，集群依然能对外提供服务。

在此基础上，为了更进一步保证用户数据的安全，TiDB 还提供了集群的**备份与恢复** (Backup & Restore, BR) 功能，作为数据安全的最后一道防线，使得集群能够免于严重的自然灾害，提供业务误操作“复原”的能力。



**以下删除线部分建议理解完1.2 之后再回来：**

~~PITR（Point-in-time recovery   恢复到指定时间点）限制：~~

- ~~仅支持恢复到全新的空集群~~
- ~~仅支持集群粒度的恢复，不支持对单个 database 或 table 的恢复~~
- ~~不支持恢复系统表中用户表和权限表的数据~~
- ~~不支持在一个集群上同时运行多个数据备份任务~~
- ~~PITR 数据**恢复**任务运行期间，不支持同时运行**日志备份**任务，也不支持通过 TiCDC **同步数据**到下游集群。~~



~~使用建议：~~

- ~~进行快照备份~~
  
  ~~推荐在业务低峰时执行集群快照数据备份，这样能最大程度地减少对业务的影响。~~
  
  ~~不推荐同时运行多个集群快照数据备份任务。不同的任务并行，不仅会导致备份的性能降低，影响在线业务，还会因为任务之间缺少协调机制造成任务失败，甚至对集群的状态产生影响。~~
- ~~进行快照恢复~~
  
  ~~BR 恢复数据时会尽可能多地占用恢复集群的资源，因此推荐恢复数据到新集群或离线集群。应避免恢复数据到正在提供服务的生产集群，否则，恢复期间会对业务产生不可避免的影响。~~

# 一、两种备份恢复

## 1.1 快照备份恢复

详见：https://docs.pingcap.com/zh/tidb/v6.5/br-snapshot-architecture



![](https://download.pingcap.com/images/docs-cn/br/br-snapshot-arch.png)



#### 1.1.1 备份

![](https://download.pingcap.com/images/docs-cn/br/br-snapshot-backup-ts.png)



#### 1.1.2 恢复

![](https://download.pingcap.com/images/docs-cn/br/br-snapshot-restore-ts.png)


## 1.2 日志备份恢复

详见：https://docs.pingcap.com/zh/tidb/v6.5/br-log-architecture

![](https://download.pingcap.com/images/docs-cn/br/br-log-arch.png)


#### 1.2.1 备份

![](https://download.pingcap.com/images/docs-cn/br/br-log-backup-ts.png)

#### 1.2.2 恢复

![](https://download.pingcap.com/images/docs-cn/br/pitr-ts.png)


# 二、建议：

### 2.1 备份：

#### 2.1.1 建议操作

- 开启**日志备份**任务后，任务会在所有 TiKV 节点上持续运行，以小批量的形式定期将 TiDB 变更数据备份到指定存储中。
- 定期执行**快照备份**，备份集群全量数据到备份存储，例如在每天零点进行集群快照备份。

### 2.1.2 性能影响

- 集群**快照数据备份**，对 TiDB 集群的影响可以保持在 20% 以下；通过合理的配置 TiDB 集群用于备份资源，影响可以降低到 10% 及更低；单 TiKV 存储节点的备份速度可以达到 50 MB/s ～ 100 MB/s，备份速度具有可扩展性；
- 单独**运行日志备份**时影响约在 5%。日志备份每隔 3～5 分钟将上次刷新后产生的变更数据记录刷新到备份存储中，可以实现低至五分钟 RPO （Recovery Point Objective）的集群容灾目标。

### 2.2 恢复：

#### 2.2.1 建议操作

- 恢复某个全量备份
  
  恢复集群**快照数据备份**：你可以在一个空集群或不存在数据冲突（相同 schema 或 table）的集群执行快照备份恢复，将该集群恢复到快照备份对应数据状态。
- 恢复到集群的历史任意时间点 (PITR)
  
  你可以指定要恢复的时间点，恢复时间点之前最近的快照数据备份，以及日志备份数据。BR 会自动判断和读取恢复需要的数据，然后将这些数据依次恢复到指定的集群。

#### 2.2.2 性能影响

- 恢复集群快照数据备份，速度可以达到单 TiKV 存储节点 100 MiB/s，恢复速度具有可扩展性；BR 只支持恢复数据到新集群，会尽可能多的使用恢复集群的资源。更详细说明请参考恢复性能和影响。
- 恢复日志备份数据，速度可以达到 30 GiB/h。更详细说明请参考 PITR 性能和影响。
