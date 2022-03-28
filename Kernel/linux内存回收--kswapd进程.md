# 脏页：

不能直接修改硬盘上的数据，而是先将数据从硬盘读入到内存的data cache，然后在内存中修改（被修改过的页称为脏数据页），最后再从内存回写到硬盘

***

# 0、1、2进程：

idle进程(PID = 0), init进程(PID = 1)和kthreadd(PID = 2)

<br/>

- idle进程由系统自动创建, 运行在内核态

idle进程其pid=0，其前身是系统创建的第一个进程，也是唯一一个没有通过fork或者kernel_thread产生的进程。完成加载系统后，演变为进程调度、交换。

<br/>

- init进程由idle通过kernel_thread创建，在内核空间完成初始化后, 加载init程序, 并最终用户空间

由0进程创建，完成系统的初始化. 是系统中所有其它用户进程的祖先进程
Linux中的所有进程都是有init进程创建并运行的。首先Linux内核启动，然后在用户空间中启动init进程，再启动其他系统进程。在系统启动完成完成后，init将变为守护进程监视系统其他进程。

- kthreadd进程由idle通过kernel_thread创建，并始终运行在内核空间, 负责所有内核线程的调度和管理

它的任务就是管理和调度其他内核线程kernel_thread, 会循环执行一个kthreadd的函数，该函数的作用就是运行kthread_create_list全局链表中维护的kthread, 当我们调用kernel_thread创建的内核线程会被加入到此链表中，因此所有的内核线程都是直接或者间接的以kthreadd为父进程

<br/>

参考：https://blog.csdn.net/gatieme/article/details/51566690

***

# 内存水位线min、low、high

![](https://pic1.zhimg.com/80/v2-f9cabbd6d35478cc14736fff5ff288b0_720w.jpg)

<br/>

在进行内存分配的时候，如果分配器（比如buddy allocator）发现当前空余内存的值低于"low"但高于"min"，说明现在内存面临一定的压力，那么在此次内存分配完成后，kswapd将被唤醒，以执行内存回收操作。在这种情况下，内存分配虽然会触发内存回收，但不存在被内存回收所阻塞的问题，两者的执行关系是异步的（之前的kswapd实现是周期性触发）。

如果内存分配器发现空余内存的值低于了"min"，说明现在内存严重不足。这里要分两种情况来讨论，一种是默认的操作，此时分配器将同步等待内存回收完成，再进行内存分配，也就是direct reclaim。还有一种特殊情况，如果内存分配的请求是带了PF_MEMALLOC标志位的，并且现在空余内存的大小可以满足本次内存分配的需求，那么也将是先分配，再回收。

那谁有这样的权利，可以在内存严重短缺的时候，不等待回收而强行分配内存呢？其中的一个人物就是kswapd啦，因为kswapd本身就是负责回收内存的.

一个zone的"low"和"high"的值都是根据它的"min"值算出来的，"low"比"min"的值大1/4左右，"high"比"min"的值大1/2左右，三者的比例关系大致是4:5:6。

<br/>

参考：https://zhuanlan.zhihu.com/p/73539328

***

# kswapd进程：

每个NUMA内存节点会有一个kswapd进程

**kswapd的目标不是回收内存，而是balance内存；**

numa内的balance是基于zone的balance。

<br/>

两个重要参数：
kswapd_max_order ：  kswapd_max_order 表示要回收内存的order，其不能小于分配内存的order
classzone_idx：是 计算的第一个合适分配内存的zone序号

<br/>

<br/>

参考：

https://blog.csdn.net/u010039418/article/details/103443581

http://linux.laoqinren.net/kernel/kswapd-thread/ (kswapd详细执行过程)

https://blog.csdn.net/weixin_33800593/article/details/90681825  （linux kswapd浅析）

***
