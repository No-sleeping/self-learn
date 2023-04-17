## 1、概念

ETCD 是**一个高可用的分布式键值数据库**，可用于服务发现。ETCD 采用**raft 一致性算法**，基于 Go语言实现。

etcd作为一个高可用键值存储系统，天生就是为集群化而设计的。由于Raft算法在做决策时需要多数节点的投票，所以etcd一般部署集群推荐奇数个节点，推荐的数量为3、5或者7个节点构成一个集群。

## 2、架构

![](https://s2.51cto.com/images/blog/202302/09100319_63e454672750c20837.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

<br/>

![](https://s2.51cto.com/images/blog/202302/09100319_63e454675aa0f35216.jpg?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

<br/>

通常，一个用户的请求发送过来，会经由 HTTP Server 转发给 Store 进行具体的事务处理，如果涉及到节点数据的修改，则交给 Raft 模块进行状态的变更、日志的记录，然后再同步给别的 etcd 节点以确认数据提交，最后进行数据的提交，再次同步。

<br/>

- HTTP Server：接受客户端发出的 API 请求以及其它 etcd 节点的同步与心跳信息请求。
- Store：用于处理 etcd 支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件理与执行等等，是etcd 对用户提供的大多数 API 功能的具体实现。
- Raft：强一致性算法的具体实现，是 etcd 的核心算法。
- WAL（Write Ahead Log，预写式日志）：是 etcd 的数据存储方式，etcd 会在内存中储存所有数据状态以及节点的索引，此外，etcd 还会通过 WAL 进行持久化存储。WAL 中，所有的数据提交前会事先记录日志。
  - Snapshot 是为了防止数据过多而进行的状态快照；
  - Entry 表示存储的具体日志内容。

<br/>

### 2.1 Http层原理

一个etcd节点运行以后，有3个通道接收外界消息，以kv数据的增删改查请求处理为例，介绍这3个通道的工作机制。

1. client的http调用：会通过注册到http模块的keysHandler的ServeHTTP方法处理。解析好的消息调用EtcdServer的Do()方法处理。
2. client的grpc调用：启动时会向grpc server注册quotaKVServer对象，quotaKVServer是以组合的方式增强了kvServer这个数据结构。grpc消息解析完以后会调用kvServer的Range、Put、DeleteRange、Txn、Compact等方法。kvServer中包含有一个RaftKV的接口，由EtcdServer这个结构实现。所以最后就是调用到EtcdServer的Range、Put、DeleteRange、Txn、Compact等方法。
3. 节点之间的grpc消息：每个EtcdServer中包含有Transport结构，Transport中会有一个peers的map，每个peer封装了节点到其他某个节点的通信方式。包括streamReader、treamWriter等，用于消息的发送和接收。streamReader中有recvc和propc队列streamReader处理完接收到的消息会将消息推到这连个队列中。由peer去处理，peer调用raftNode的Process方法处理消息。

<br/>

### 2.2 EtcdServer层

对于客户端消息，调用到EtcdServer处理时，一般都是先注册一个等待队列，调用node的Propose方法，然后用等待队列阻塞等待消息处理完成。Propose方法会往propc队列中推送一条MsgProp消息。

对于节点间的消息，raftNode的Process是直接调用node的step方法，将消息推送到node的recvc或者propc队列中。可以看到，所有消息这时候都到了node结构中的recvc队列或者propc队列中。

<br/>

## 3、 Raft协议算法原理

Raft协议采用分治的思想，把分布式协同的问题分为3个问题：

- 选举： 一个新的集群启动时，或者老的leader故障时，会选举出一个新的leader。
- 日志同步： leader必须接受客户端的日志条目并且将他们同步到集群的所有机器。
- 安全： 保证任何节点只要在它的状态机中生效了一条日志，就不会在相同的key上生效另一条日志条目。

一个Raft集群一般包含数个节点，典型的是5个，这样可以承受其中2个节点故障。每个节点实际上就是维护一个状态机，节点在任何时候都处于以下三个状态中的一个。

- leader：负责日志的同步管理，处理来自客户端的请求，与Follower保持这heartBeat的联系
- follower：刚启动时所有节点为Follower状态，响应Leader的日志同步请求，响应Candidate的请求，把请求到Follower的事务转发给Leader
- candidate：负责选举投票，Raft刚启动时由一个节点从Follower转为Candidate发起选举，选举出Leader后从Candidate转为Leader状态

![](https://s2.51cto.com/images/blog/202302/09100319_63e454679e01a62716.jpg?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

<br/>

节点启动以后，首先都是follower状态，在follower状态下，会有一个选举超时时间的计时器(这个时间是在配置的超时时间基础上加一个随机的时间得来的)。

如果在这个时间内没有收到leader发送的心跳包，则节点状态会变成candidate状态，也就是变成了候选人，候选人会循环广播选举请求，如果超过半数的节点同意选举请求，则节点转化为leader状态。

如果在选举过程中，发现已经有了leader或者有更高的任期值的选举信息，则自动变成follower状态。

处于leader状态的节点如果发现有更高任期值的leader存在，则也是自动变成follower状态。

<br/>

Raft把时间划分为任期(Term)(如下图所示)，任期是一个递增的整数，一个任期是从开始选举leader到leader失效的这段时间。

有点类似于一届总统任期，只是它的时间是不一定的，也就是说只要leader工作状态良好，它可能成为一个独裁者，一直不下台。

![](https://s2.51cto.com/images/blog/202302/09100319_63e45467c510c19031.jpg?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

<br/>

<br/>

## 4、etcd的特点

- 简单：基于HTTP+JSON的API让你用curl命令就可以轻松使用。
- 安全：可选SSL客户认证机制。
- 快速：每个实例每秒支持一千次写操作。
- 可信：使用Raft算法充分实现了分布式。

<br/>

## 5、etcd相关名词

- Raft：etcd所采用的保证分布式系统强一致性的算法。
- Node：一个Raft状态机实例。
- Member： 一个etcd实例。它管理着一个Node，并且可以为客户端请求提供服务。
- Cluster：由多个Member构成可以协同工作的etcd集群。
- Peer：对同一个etcd集群中另外一个Member的称呼。
- Client： 向etcd集群发送HTTP请求的客户端。
- WAL：预写式日志，etcd用于持久化存储的日志格式。
- snapshot：etcd防止WAL文件过多而设置的快照，存储etcd数据状态。
- Proxy：etcd的一种模式，为etcd集群提供反向代理服务。
- Leader：Raft算法中通过竞选而产生的处理所有数据提交的节点。
- Follower：竞选失败的节点作为Raft中的从属节点，为算法提供强一致性保证。
- Candidate：当Follower超过一定时间接收不到Leader的心跳时转变为Candidate开始Leader竞选。
- Term：某个节点成为Leader到下一次竞选开始的时间周期，称为一个Term。
- Index：数据项编号。Raft中通过Term和Index来定位数据。

<br/>

---

https://blog.51cto.com/u_13643065/6112220   （Etcd——基础架构和相关原理）

https://cloud.tencent.com/developer/article/2129598  （Etcd基础学习之架构及工作原理）
