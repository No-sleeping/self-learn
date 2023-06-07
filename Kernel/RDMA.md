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

RDMA （ Remote Direct Memory Access，远程直接地址访问技术 ） 是一种新的内存访问技术，可以让服务器直接高速读写其他服务器的内存数据，而不需要经过操作系统/CPU耗时的处理。

![](http://mmbiz.qpic.cn/mmbiz_png/GOHsECYibE4RPxx7S7GKX9fsftyFicYHnMibCwEgIDT4VZRXsU08goVrhDmImPNXGvKD9b8sgP574rT8osH1OsWVA/640?mprfK=http%3A%2F%2Fmp.weixin.qq.com%2F)

<br/>

### 1.1 优势

- Zero Copy（零拷贝）： 无需将数据拷贝到操作系统内核态并处理数据包头部的过程，传输延迟会显著减小。
- Kernel Bypass（内核旁路）和Protocol Offload（协议卸载）： 不需要操作系统内核参与，数据通路中没有繁琐的处理报头逻辑，不仅会使延迟降低，而且也大大节省了CPU的资源。
- CPU Offload（CPU Bypass）：无需CPU干预，应用程序可以访问远程主机内存而不消耗远程主机中的任何CPU（这里面可能有歧义，应该还会通知CPU的）。远程主机内存能够被读取而不需要远程主机上的进程（或CPU)参与。远程主机的CPU的缓存(cache)不会被访问的内存内容所填充。
- 异步接口：在RDMA中，提供的所有接口都是异步的通信接口，这样在编程的时候，可以更加便利的实现计算和通信的分离。

##  2、RDMA  通信协议

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

---

https://maimai.cn/article/detail?fid=1761674830&efid=T_YZaGxuBX_GDevAO1KHTA   （通俗易懂）

https://blog.csdn.net/u011458874/article/details/121602188  （初识RDMA技术——RDMA概念，特点，协议，通信流程）

https://zhuanlan.zhihu.com/p/361740115  （较全）
