> wrk is a modern HTTP benchmarking tool capable of generating significant load when run on a single multi-core CPU. It combines a multithreaded design with scalable event notification systems such as epoll and kqueue

wrk是一种现代HTTP基准测试工具，能够在单个多核CPU上运行时产生显著的负载。它将多线程设计与可伸缩的**事件通知系统（如epoll和kqueue）**结合在一起。

<br/>

一、事件驱动模型（https://baike.baidu.com/item/%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B/1419787?fr=aladdin）

1. select
   1. 创建读、写、异常三个描述符集合(fd_set)
   2. > int select(int nfds ,
fd_set* readfds ,
fd_set* writefds ,
fd_set* exceptfds,
const struct timeval* timeout
   3. 用户态创建的fd_set会全部copy到内核态之后，内核态再次轮询。所以会有两次copy(创建一个事件列表，然后把这个列表发给内核，返回的时候，再去轮询检查这个列表)
2. poll
   1. > int poll(struct pollfd *fds, nfds_t nfds, int timeout);
   2. 与select 区别只有一个fd_set(先创建一个关注事件的描述符的集合，然后再去等待这些事件发生，然后再轮询描述符集合，检查有没有事件发生，如果有，就进行处理）
3. epoll
   1. 把描述符列表交给内核，一旦有事件发生，内核把发生事件的描述符列表通知给进程，这样就避免了轮询整个描述符列表
   2. 创建一个epoll描述符，调用epoll_create()来完成，epoll_create()有一个整型的参数size，用来告诉内核，要创建一个有size个描述符的事件列表（集合）
   3. 给描述符设置所关注的事件，并把它添加到内核的事件列表中去，这里需要调用epoll_ctl()来完成
   4. 等待内核通知事件发生，得到发生事件的描述符的结构列表，该过程由epoll_wait()完成。得到事件列表后，就可以进行事件处理了.
   5. 水平触发：Level Triggered(LT), 在这种情况下，epoll和poll类似，但处理速度上可能比poll快。在这种情况下，只要有数据没有读、写完，调用epoll_wait()的时候，就会有事件被触发。
   6. 边界触发：Edge Triggered(ET)，在这种情况下，事件是由数据到达边界触发的。所以要在处理读、写的时候，要不断的调用read/write，直到它们返回EAGAIN，然后再去epoll_wait(),等待下次事件的发生。这种方式适用要遵从下面的原则：
a. 使用非阻塞的I/O；b.直到read/write返回EAGAIN时，才去等待下一次事件的发生。

二、阻塞、非阻塞IO

<br/>

阻塞IO

![](https://img-blog.csdnimg.cn/d08289abdf4f4cc2a65817daaf67fba6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAU2Fuc2lwaQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

非阻塞IO

![](https://img-blog.csdnimg.cn/5b0ef41f29664a6d9dc7dfe7fe798867.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAU2Fuc2lwaQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
