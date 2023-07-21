## 0、背景

计算、存储和网络，是推动数据中心发展的三驾马车。

计算随着CPU、GPU和FPGA的发展，算力得到了极大的提升。 存储随着闪存盘（SSD）的引入，数据存取时延已大幅降低。

但是，网络的发展明显滞后，传输时延高，逐渐成为了数据中心高性能的瓶颈。

在数据中心内，70%的流量为东西向流量（服务器之间的流量）。 这些流量，一般为数据中心进行高性能分布式并行计算时的过程数据流，通过TCP/IP网络传输。

如果服务器之间的TCP/IP传输速率提升了，数据中心的性能自然也会跟着提升。

<br/>

在数据中心，服务器A向服务器B发送数据的过程如下：

![](http://mmbiz.qpic.cn/mmbiz_png/GOHsECYibE4RPxx7S7GKX9fsftyFicYHnMDCxzgsNw7lVbRdY0m7YLv1gAwnxMoVUyJCxRWewNlCiczQ6d6AzwDBw/640?mprfK=http%3A%2F%2Fmp.weixin.qq.com%2F)

从数据传输的过程可以看出，数据在服务器的**Buffer内多次拷贝**，在操作系统中需要添加/卸载TCP、IP报文头，这些操作既增加了数据传输时延，又消耗了大量的CPU资源，无法很好得满足高性能计算的需求。

## 1、RDMA

RDMA(Remote Direct Memory Access)技术全称远程直接内存访问，就是为了解决网络传输中客户端与服务器端数据处理的延迟而产生的。它将数据直接从一台计算机的内存传输到另一台计算机，无需双方操作系统的介入。这允许高吞吐、低延迟的网络通信，尤其适合在大规模并行计算机集群中使用。RDMA通过网络把资料直接传入计算机的内存中，将数据从一个系统快速移动到远程系统内存中，而不对操作系统造成任何影响，这样就不需要用到多少计算机的处理能力。它消除了数据包在用户空间和内核空间复制移动和上下文切换的开销，因而能解放内存带宽和CPU周期用于改进应用系统性能。（注：原始的TCP/IP通过封装数据包的形式，通过OS与协议栈进行解析、处理数据，主机开销变多）

![](https://img-blog.csdnimg.cn/2020062410534438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE0OTQ1MzI3,size_16,color_FFFFFF,t_70)

<br/>

### 1.1 优势

- Zero Copy（零拷贝）： 无需将数据拷贝到操作系统内核态并处理数据包头部的过程，传输延迟会显著减小。
- Kernel Bypass（内核旁路）和Protocol Offload（协议卸载）： 不需要操作系统内核参与，数据通路中没有繁琐的处理报头逻辑，不仅会使延迟降低，而且也大大节省了CPU的资源。
- CPU Offload（CPU Bypass）：无需CPU干预，应用程序可以访问远程主机内存而不消耗远程主机中的任何CPU（这里面可能有歧义，应该还会通知CPU的）。远程主机内存能够被读取而不需要远程主机上的进程（或CPU)参与。远程主机的CPU的缓存(cache)不会被访问的内存内容所填充。
- 异步接口：在RDMA中，提供的所有接口都是异步的通信接口，这样在编程的时候，可以更加便利的实现计算和通信的分离。

##  2、RDMA  通信协议

RDMA扩展网卡的能力，不需要CPU参与，就可以实现在两台通信的主机间完成内存数据复制操作。RDMA提供了三种技术规范实现方式，分别是IB (Infiniband), iWARP (Internet Wide Area RDMA Protocol) 和RoCE (RDMA over Converged Ethernet)。三种实现都支持IBTA (InfiniBand Trade Association) 制定的RDMA Verbs原语和数据类型，提供统一的业务编程接口供用户使用，达到业务无缝切换。

 IB需要专用的IB网卡和IB交换机。性能优，但网卡和交换机的价格昂贵，兼容性差。IWARP 技术栈需要通用以太网交换机和支持iWARP功能的以太网卡。报文依赖TCP连接，TCP连接需要占用内核资源,市场认可度低于RoCE。RoCE技术栈只需要通用以太网网卡和交换机，利用PFC(Priority-based Flow Control )和ECN (Explicit Congestion Notification) 拥塞控制算法实现无损传输。其综合性能较好，兼容性较优，价格也很普惠。RoCE市场认可度很高，用户可以根据自己的使用场景和实际需求选择对应的产品。

![](https://img2020.cnblogs.com/blog/2325714/202103/2325714-20210316110620305-878895229.png)

<br/>

- InfiniBand

InfiniBand（IB）是一种服务器和存储器的互联技术，它具有高速、低延迟、低CPU负载、高效率和可扩展的特性。InfiniBand的关键特性之一是它天然地支持远程直接内存访问（RDMA）。InfiniBand能够让服务器与服务器之间、服务器与存储设备之间的数据传输不需要主机CPU的参与。

InfiniBand使用I/O通道进行数据传输，每个I/O通道提供虚拟的NIC或HCA语义。**InfiniBand使用同轴电缆和光纤进行连接**。

- RoCE
  
  RoCE是基于以太网（Ethernet）的RDMA技术标准，它也是由IBTA组织指定的。RoCE为以太网提供了RDMA语义，并不需要复杂低效的TCP传输（IWARP需要）。
  
  RoCE是现在最有效的以太网低延迟方案。它消耗很少的CPU负载，在数据中心桥接以太网中利用优先流控制（PFC）来达到网络的无损连接。
  
  RoCE 有两个版本，RoCE v1是一种链路层协议，允许在同一个广播域下的任意两台主机直接访问。
  
  **RoCE v2**是一种Internet层协议，即可以实现路由功能。虽然RoCE协议这些好处都是基于融合以太网的特性，但是RoCE协议也可以使用在传统以太网网络或者非融合以太网络中。
- iWARP
  
  iWARP也是一个允许在TCP上执行RDMA的网络协议。IB和RoCE中存在的功能在iWARP中不受支持。它支持在标准以太网基础设施（交换机）上使用RDMA

<br/>

### 2.2 RDMA的工作过程与细节

#### RDMA 的工作过程如下:

（1）当一个应用执行RDMA 读或写请求时，不执行任何数据复制.在不需要任何内核内存参与的条件下，RDMA 请求从运行在用户空间中的应用中发送到本地NIC( 网卡)。

（2）NIC 读取缓冲的内容，并通过网络传送到远程NIC。

（3）在网络上传输的RDMA 信息包含目标虚拟地址、内存钥匙和数据本身请求，既可以完全在用户空间中处理（通过轮询用户级完成排列），又或者在应用一直睡眠到请求完成时的情况下通过系统中断处理。RDMA 操作使应用可以从一个远程应用的内存中读数据或向这个内存写数据。

（4）目标NIC 确认内存钥匙，直接将数据写人应用缓存中，用于操作的远程虚拟内存地址包含在RDMA 信息中。

#### RDMA操作细节

RDMA提供了基于消息队列的点对点通信，每个应用都可以直接获取自己的消息，无需操作系统和协议栈的介入。

消息服务建立在通信双方的本地与远端应用之间创建的Channel-IO连接之上。

每对QP由**Send Queue（SQ）和Receive Queue（RQ）**构成，这些队列中管理着各种类型的消息。QP会被映射到应用的虚拟地址空间，使得应用直接通过它访问RNIC网卡。除了QP描述的两种基本队列之外，RDMA还提供一种队列**Complete Queue（CQ），CQ用来知会用户WQ上的消息已经被处理完。**

RDMA提供了一套软件传输接口，方便用户创建传输请求Work Request(WR），WR中描述了应用希望传输到Channel对端的消息内容，WR通知QP中的某个队列Work Queue(WQ)。在WQ中，用户的WR被转化为Work Queue Element（WQE）的格式，等待RNIC的异步调度解析，并从WQE指向的Buffer中拿到真正的消息发送到Channel对端。

RDMA有单边读、单边写，双边操作。

<br/>

<br/>

## 3、 特性

### 3.1 PFC

![](https://img-blog.csdnimg.cn/img_convert/ba069dd5316d0eb074abcc49137fa680.png)

如下图所示，端口G0/1和G0/2以1Gbps速率转发报文时，端口F0/1将发生拥塞。为避免报文丢失，开启端口G0/1和G0/2的Flow Control功能。

当F0/1在转发报文出现拥塞时，交换机B会在端口缓冲区中排队报文，当拥塞超过一定阈值时，端口G0/2向G0/1发PAUSE帧，通知G0/1暂时停止发送报文。

G0/1接收到PAUSE帧后暂时停止向G0/2发送报文。暂停时间长短信息由PAUSE帧所携带。交换机A会在这个超时范围内等待，或者直到收到一个Timeout值为0的控制帧后再继续发送。

###  3.2  ECN

![](https://img-blog.csdnimg.cn/img_convert/d641b52409b4edf5f484487ccf8eb883.png)

<br/>

① 发送端发送的IP报文标记支持ECN(10); 

② 交换机在队列拥塞情况下收到该报文，将ECN字段修改为11并发出，网络中其他交换机将透传;

③ 接收端收到ECN为11的报文发现拥塞，正常处理该报文;

④ 接收端产生拥塞通告，每ms级发送一个CNP(Congestion Notification Packets)报文，ECN字段为01，要求报文不能被网络丢弃。接收端对多个被ECN标记为同一个QP的数据包发送一个单个CNP即可(格式规定见下图);--（即对同一个QP的数据发送同一个CNP即可）

⑤ 交换机收到CNP报文后正常转发该报文;

⑥ 发送端收到ECN标记为01的CNP报文解析后对相应的流(对应启用ECN的QP)应用速率限制算法。

<br/>

![](https://img-blog.csdnimg.cn/20210313150908906.png)

<br/>

---

https://www.h3c.com/cn/Service/Document_Software/Document_Center/Home/Switches/00-Public/Learn_Technologies/White_Paper/RDMA_Tech_White_Paper-6W100/  （RDMA技术白皮书）

https://blog.csdn.net/bandaoyu/article/details/115346857 （无损网络和PFC（基于优先级的流量控制）|ECN）

https://maimai.cn/article/detail?fid=1761674830&efid=T_YZaGxuBX_GDevAO1KHTA   （通俗易懂）

https://blog.csdn.net/u011458874/article/details/121602188  （初识RDMA技术——RDMA概念，特点，协议，通信流程）

https://zhuanlan.zhihu.com/p/361740115  （较全）
