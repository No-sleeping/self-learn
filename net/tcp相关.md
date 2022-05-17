**序列号**：在建立连接时由计算机生成的随机数作为其初始值，通过 SYN 包传给接收端主机，每发送一次数据，就「累加」一次该「数据字节数」的大小。用来解决网络包乱序问题。

**确认应答号**：指下一次「期望」收到的数据的序列号，发送端收到这个确认应答以后可以认为在这个序号以前的数据都已经被正常接收。用来解决丢包的问题。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jZG4uanNkZWxpdnIubmV0L2doL3hpYW9saW5jb2Rlci9JbWFnZUhvc3QyLyVFOCVBRSVBMSVFNyVBRSU5NyVFNiU5QyVCQSVFNyVCRCU5MSVFNyVCQiU5Qy9UQ1AtJUU0JUI4JTg5JUU2JUFDJUExJUU2JThGJUExJUU2JTg5JThCJUU1JTkyJThDJUU1JTlCJTlCJUU2JUFDJUExJUU2JThDJUE1JUU2JTg5JThCLzYuanBn?x-oss-process=image/format,png)

<br/>

**tcp三次握手详解：**

> https://www.cnblogs.com/liqianlong/p/8690088.html

### 等待时间为什么是2MSL？

首先说明什么是MSL，MSL是Maximum Segment Lifetime的缩写，译为报文最大生存时间，也就是任何报文在网络上存活的最大时间，一旦超过该时间，报文就会被丢弃。2MSL也就是指的2倍MSL的时间。

为什么是2倍呢？
主动断开的一侧为A，被动断开的一侧为B。
第一个消息：A发FIN
第二个消息：B回复ACK
第三个消息：B发出FIN此时此刻：B单方面认为自己与A达成了共识，即双方都同意关闭连接。此时，B能释放这个TCP连接占用的内存资源吗？不能，B一定要确保A收到自己的ACK、FIN。所以B需要静静地等待A的第四个消息的到来：
第四个消息：A发出ACK，用于确认收到B的FIN
当B接收到此消息，即认为双方达成了同步：双方都知道连接可以释放了，此时B可以安全地释放此TCP连接所占用的内存资源、端口号。
所以被动关闭的B无需任何wait time，直接释放资源。
但，A并不知道B是否接到自己的ACK，A是这么想的：
1）如果B没有收到自己的ACK，会超时重传FIN,那么A再次接到重传的FIN，会再次发送ACK
2）如果B收到自己的ACK，也不会再发任何消息，包括ACK
无论是1还是2，A都需要等待，要取这两种情况等待时间的最大值，以应对最坏的情况发生，这个最坏情况是：

去向ACK消息最大存活时间（MSL) + 来向FIN消息的最大存活时间(MSL)。

这恰恰就是2MSL( Maximum Segment Life)。

等待2MSL时间，A就可以放心地释放TCP占用的资源、端口号，此时可以使用该端口号连接任何服务器。同时也能保证网络中老的链接全部消失。

<br/>

---

# tcp三次握手异常场景分析

1. TCP 第一次握手的 SYN 包超时重传最大次数是由 tcp_syn_retries 指定（默认值是 5 次）
   1. 通过实验一的实验结果，我们可以得知，当客户端发起的 TCP 第一次握手 SYN 包，在超时时间内没收到服务端的 ACK，就会在超时重传 SYN 数据包，每次超时重传的 RTO 是翻倍上涨的，直到 SYN 包的重传次数到达 tcp_syn_retries 值后，客户端不再发送 SYN 包。
2. TCP 第二次握手的 SYN、ACK 包超时重传最大次数是由 tcp_synack_retries 指定（默认值是 5 次）
   1. 通过实验二的实验结果，我们可以得知，当 TCP 第二次握手 SYN、ACK 包丢了后，客户端 SYN 包会发生超时重传，服务端 SYN、ACK 也会发生超时重传。
   2. 客户端 SYN 包超时重传的最大次数，是由 tcp_syn_retries 决定的，默认值是 5 次；服务端 SYN、ACK 包时重传的最大次数，是由 tcp_synack_retries 决定的，默认值是 5 次。
3. 那 TCP 建立连接后的数据包最大超时重传次数是由tcp_retries2指定 （默认值是 15 次）
   1. 在建立 TCP 连接时，如果第三次握手的 ACK，服务端无法收到，则服务端就会短暂处于 SYN_RECV 状态，而客户端会处于 ESTABLISHED 状态。
   2. 由于服务端一直收不到 TCP 第三次握手的 ACK，则会一直重传 SYN、ACK 包，直到重传次数超过 tcp_synack_retries 值（默认值 5 次）后，服务端就会断开 TCP 连接。
   3. 客户端则会有两种情况：
      1. 如果客户端没发送数据包，一直处于 ESTABLISHED 状态，然后经过 2 小时 11 分 15 秒才可以发现一个「死亡」连接，于是客户端连接就会断开连接。
      2. 如果客户端发送了数据包，一直没有收到服务端对该数据包的确认报文，则会一直重传该数据包，直到重传次数超过 tcp_retries2 值（默认值 15 次）后，客户端就会断开 TCP 连接。

<br/>

# 为什么是三次握手，不是两次，四次

1. 三次握手才可以阻止重复历史连接的初始化（主要原因）
   1. 一个「旧 SYN 报文」比「最新的 SYN 」 报文早到达了服务端；
   2. 那么此时服务端就会回一个 SYN + ACK 报文给客户端；
   3. 客户端收到后可以根据自身的上下文，判断这是一个历史连接（序列号过期或超时），那么客户端就会发送 RST 报文给服务端，表示中止这一次连接。
2. 三次握手才可以同步双方的初始序列号
   1. 序列号作用
      1. 接收方可以去除重复的数据；
      2. 接收方可以根据数据包的序列号按序接收；
      3. 可以标识发送出去的数据包中， 哪些是已经被对方收到的（通过 ACK 报文中的序列号知道）
3. 三次握手才可以避免资源浪费
