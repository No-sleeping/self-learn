## 优点

- 通过UIO技术将报文拷贝到应用空间处理，规避不必要的内存拷贝和系统调用，便于快速迭代优化。
- 通过大页内存HUGEPAGE，降低cache miss（访存开销），利用内存多通道交错访问提高内存访问有效带宽，即提高命中率，进而提高cpu访问速度。
- 通过CPU亲和性，绑定网卡和线程到固定的core，减少cpu任务切换。特定任务可以被指定只在某个核上工作，避免线程在不同核间频繁切换，保证更多的cache命中。
- 通过无锁队列，减少资源竞争。cache行对齐，预取数据，多元数据批量操作。
- 通过轮询可在包处理时避免中断上下文切换的开销。

> - 用户态模式的PMD驱动，去除中断，避免内核态和用户态内存拷贝，减少系统开销，从而提升I/O吞吐能力
> - 用户态有一个好处，一旦程序崩溃，不至于导致内核完蛋，带来更高的健壮性
> - HugePage，通过更大的内存页（如1G内存页），减少TLB（Translation Lookaside Buffer，即快表） Miss，Miss对报文转发性能影响很大
> - 多核设备上创建多线程，每个线程绑定到独立的物理核，减少线程调度的开销。同时每个线程对应着独立免锁队列，同样为了降低系统开销
> - 向量指令集，提升CPU流水线效率，降低内存等待开销

![](https://img-blog.csdn.net/20180307221439106?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvb2xkYm95XzE5ODM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

> - PMD：Pool Mode Driver，轮询模式驱动，通过非中断，以及数据帧进出应用缓冲区内存的零拷贝机制，提高发送/接受数据帧的效率
> - 流分类：Flow Classification，为N元组匹配和LPM（最长前缀匹配）提供优化的查找算法
> - 环队列：Ring Queue，针对单个或多个数据包生产者、单个数据包消费者的出入队列提供无锁机制，有效减少系统开销
> - MBUF缓冲区管理：分配内存创建缓冲区，并通过建立MBUF对象，封装实际数据帧，供应用程序使用
> - EAL：Environment Abstract Layer，环境抽象（适配）层，PMD初始化、CPU内核和DPDK线程配置/绑定、设置HugePage大页内存等系统初始化

<br/>

DPDK将网卡接收队列分配给某个CPU核，该队列收到的报文都交给该核上的DPDK线程处理。存在两种方式将数据包发送到接收队列之上：

- RSS（Receive Side Scaling，接收方扩展）机制：根据关键字，比如根据UDP的四元组<srcIP><dstIP><srcPort><dstPort>进行哈希
- Flow Director机制：可设定根据数据包某些信息进行精确匹配，分配到指定的队列与CPU核

![](https://img-blog.csdn.net/20180307224443703?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvb2xkYm95XzE5ODM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 优点详解

### 1、UIO（Linux Userspace I/O）

![](https://img-blog.csdnimg.cn/20190911222323969.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FwZUxpZmU=,size_16,color_FFFFFF,t_70)

- 在系统加载igb_uio驱动后，每当有网卡和igb_uio驱动进行绑定时， 就会在/dev目录下创建一个uio设备,例如/dev/uio1。uio设备是一个接口层，用于将**pci网卡的内存空间以及网卡的io空间暴露给应用层**。通过这种方式，应用层访问uio设备就相当于访问网卡。

- 具体来说，当有网卡和uio驱动绑定时，被内核加载的igb_uio驱动，** 会将pci网卡的内存空间，io空间保存在uio目录下，**例如/sys/class/uio/uio1/maps文件中，**同时也会保存到pci设备目录下的uio文件中。**

- **这样应用层就可以访问这两个文件中的任意一个文件里面保存的地址空间，然后通过mmap将文件中保存网卡的物理内存映射成虚拟地址， 应用层访问这个虚拟地址空间就相当于访问pci设备。**

- 内核态驱动igb_uio，用于将pci网卡的内存空间，io空间暴露给应用层，供应用层访问，同时会处理在网卡的硬件中断（控制中断而不是数据中断）

<br/>

### 2、用户空间轮询模式（PMD）

DPDK用户空间的轮询模式驱动：用户空间驱动使得应用程序不需要经过linux内核就可以访问网络设备卡。

网卡设备可以通过DMA方式将数据包传输到事先分配好的缓冲区，这个缓冲区位于用户空间，应用程序通过不断轮询的方式可以读取数据包并在原地址上**直接处理，不需要中断**，而且也省去了内核到应用层的数据包拷贝过程。

<br/>

### 3、大页内存

Linux操作系统通过查找TLB来实现快速的虚拟地址到物理地址的转化。

由于TLB是一块高速缓冲cache，容量比较小，容易发生没有命中。当没有命中的时候，会触发一个中断，然后会访问内存来刷新页表，这样会造成比较大的时延，降低性能。Linux操作系统的页大小只有4K，所以当应用程序占用的内存比较大的时候，会需要较多的页表，开销比较大，而且容易造成未命中。

相比于linux系统的4KB页，Intel DPDK缓冲区管理库提供了**Hugepage大页内存，大小有2MB和1GB页面两种**，可以得到明显性能的提升，因为采用大页内存的话，可以需要更少的页，从而需要更少的TLB，这样就减少了虚拟页地址到物理页地址的转换时间。

<br/>

### 4、CPU亲和性

CPU的亲和性（CPU affinity），它是多核CPU发展的结果。

随着核心的数量越来越多，为了提高程序工作的效率必须使用多线程。但是随着CPU的核心的数目的增长，Linux的核心间的调度和共享内存争用会严重影响性能。**利用Intel DPDK的CPU affinity可以将各个线程绑定到不同的cpu，可以省去来回反复调度带来的性能上的消耗。**

在一个多核处理器的机器上，每个CPU核心本身都存在自己的缓存，缓冲区里存放着线程使用的信息。如果线程没有绑定CPU核，那么线程可能被Linux系统调度到其他的CPU上，这样的话，CPU的cache命中率就降低了。利用CPU的affinity技术，一旦线程绑定到某个CPU后，线程就会一直在指定的CPU上运行，操作系统不会将其调度到其他的CPU上，节省了调度的性能消耗，从而提升了程序执行的效率。

多核轮询模式：多核轮询模式有两种，分别是IO独占式和流水线式。

**IO独占式**是指每个核独立完成数据包的接收、处理和发送过程，核之间相互独立，其优点是其中一个核出现问题时不影响其他核的数据收发。流水线式则采用多核合作的方式处理数据包，数据包的接收、处理和发送由不同的核完成。流水线式适合面向流的数据处理，其优点是可对数据包按照接收的顺序有序进行处理，缺点是当某个环境（例如接收）所涉及的核出现阻塞，则会造成收发中断。

IO独占式多核轮询模式中每个网卡队列只分配给一个逻辑核进行处理。每个逻辑核给所接管的网卡分别分配一个发送队列和一个接收队列，并且独立完成数据包的接收、处理和发送的过程，核与核之间相互独立。系统数据包的处理由多个逻辑核同时进行，每个网卡的收发包队列只能由一个逻辑核提供。当数据包进入网卡的硬件缓存区，用户空间提供的网卡驱动通过轮询得知网卡收到数据包，从硬件缓冲区中取出数据包，并将数据包存入逻辑核提供的收包队列中，逻辑核取出收包队列中的数据包进行处理，处理完毕后将数据包存入逻辑核提供的发包队列，然后由网卡驱动取出发往网卡，最终发送到网络中。

![](https://img-blog.csdnimg.cn/20200418224304601.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwODE3MzI3,size_16,color_FFFFFF,t_70)

### 5、内存池和无锁环形缓存管理

此外Intel DPDK将库和API优化成了无锁，比如无锁队列，可以防止多线程程序发生死锁。然后对缓冲区等数据结构进行了**cache对齐。**如果没有cache对齐，则可能在内存访问的时候多读写一次内存和cache。

内存池缓存区的申请和释放采用的是生产者-消费者模式无锁缓存队列进行管理，避免队列中锁的开销，在缓存区的使用过程中提高了缓冲区申请释放的效率。

<br/>

### 6、网络存储优化

![](https://img-blog.csdnimg.cn/20200528173514303.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwODE3MzI3,size_16,color_FFFFFF,t_70)

<br/>

## 对比图

<br/>

![](https://img-blog.csdnimg.cn/20200528171614482.png)

<br/>

![](https://img-blog.csdnimg.cn/20200528171643658.png)

<br/>

<br/>

![](https://img-blog.csdnimg.cn/20200528171811665.png)

***

参考：

https://blog.csdn.net/oldboy_1983/article/details/79474750   （DPDK的基本原理）

https://blog.csdn.net/qq_20817327/article/details/105587309 （ 原理---较全）

https://blog.csdn.net/wangquan1992/article/details/104051163  （dpdk uio驱动实现）
