### BPF
数据包达到网络接口，连接层的设备通常会把它发送到系统协议栈。
但是当BPF监听端口的时候，就会发送给BPF。
BPF又把这些 数据包给到进程过滤器。
这些用户定义的过滤器，又决定哪些数据包留下，或者每个包留下多少字节。
BPF会复制每个进程过滤器相关的数据包到buffer。



### eBPF

eBPF 分为用户空间程序和内核程序两部分：

用户空间程序负责加载 BPF 字节码至内核，如需要也会负责读取内核回传的统计信息或者事件详情；
内核中的 BPF 字节码负责在内核中执行特定事件，如需要也会将执行的结果通过 maps 或者 perf-event 事件发送至用户空间；


用户空间程序与内核中的 BPF 字节码交互的流程主要如下：

我们可以使用 LLVM 或者 GCC 工具将编写的 BPF 代码程序编译成 BPF 字节码；
然后使用加载程序 Loader 将字节码加载至内核；内核使用验证器（verfier） 组件保证执行字节码的安全性，以避免对内核造成灾难，在确认字节码安全后将其加载对应的内核模块执行；


### 区别

|维度|cBPF|eBPF|
|--|--|--|
|内核版本|Linux 2.1.75（1997年）|Linux 3.18（2014年）[4.x for kprobe/uprobe/tracepoint/perf-event]|
|寄存器数目|2个：A, X|10个： R0–R9, 另外 R10 是一个只读的帧指针 * R0 - eBPF 中内核函数的返回值和退出值 * R1 - R5 - eBF 程序在内核中的参数值 * R6 - R9 - 内核函数将保存的被调用者callee保存的寄存器 * R10 -一个只读的堆栈帧指针|
|寄存器宽度|32位|64位|
|存储|16 个内存位: M[0–15]|512 字节堆栈，无限制大小的 “map” 存储|
|限制的内核调用|非常有限，仅限于 JIT 特定|有限，通过 bpf_call 指令调用|
|目标事件|数据包、 seccomp-BPF|数据包、内核函数、用户函数、跟踪点 PMCs 等|

---
参考：

https://www.tigera.io/blog/introducing-the-calico-ebpf-dataplane/   （Introducing the Calico eBPF dataplane）

https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables （Why is the kernel community replacing iptables with BPF?）

http://www.dockone.io/article/10484   （腾讯TKE用eBPF）

https://cilium.io/blog/2021/05/11/cni-benchmark （CNI Benchmark: Understanding Cilium Network Performance）

https://www.tcpdump.org/papers/bpf-usenix93.pdf   （The BSD Packet Filter:A New Architecture for User-level Packet Capture）

https://lwn.net/Articles/740157/  （A thorough introduction to eBPF）

https://davidlovezoe.club/wordpress/archives/1122 （LINUX超能力BPF技术介绍及学习分享）

https://github.com/DavadDi/bpf_study#23-ebpf-%E7%9A%84%E9%99%90%E5%88%B6 （ebpf技术简介–较全）

国内大厂 eBPF 实践总结：

eBPF 在网易轻舟云原生的应用实践   https://www.infoq.cn/article/OVCVwQijztA7JlexgDOc

性能提升40%: 腾讯 TKE 用 eBPF绕过 conntrack 优化 K8s Service   https://mp.weixin.qq.com/s?__biz=MzI5ODQ2MzI3NQ==&mid=2247491111&idx=2&sn=db348d6f13e1df4b3b9aba2dce0ba0e4&chksm=eca42763dbd3ae757530f6922ca1748736e42eb863e01076e94622c81be542e5582c9678874b&scene=27#wechat_redirect

字节跳动：eBPF 技术实践：高性能 ACL   https://www.infoq.cn/article/Tc5Bugo5vBAkyaRb5CCU

阿里：eBPF Internal：Instructions and Runtime   https://www.infoq.cn/article/c6t2IL23O6EbdQgUpQhb

使用 ebpf 深入分析容器网络 dup 包问题   https://mp.weixin.qq.com/s?__biz=MzI5ODQ2MzI3NQ==&mid=2247488831&idx=1&sn=3da3a976439d0134e3789a3e035ea1f0&chksm=eca42c7bdbd3a56d35c482d07798ee9d48a2f1103724f78f0634953ab33d8bd1ab9700190fb6&scene=27#wechat_redirect

eBay 云计算“网”事：网络超时篇   https://www.infoq.cn/article/JmCbkA0XX9NqrcX6loIo

字节跳动容器化场景下的性能优化实践   https://www.infoq.cn/article/mu-1bFHNmrdd0kybgPXx

PingCAP Libbpf-tools —— 让 Tracing 工具身轻如燕   https://mp.weixin.qq.com/s/-3QRMu1aQbGxaF_JQY353w


