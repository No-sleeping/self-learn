## 1、iptables
![数据经过防火墙的流程](https://github.com/No-sleeping/self-learn/blob/main/images/kubernetes/%E6%95%B0%E6%8D%AE%E7%BB%8F%E8%BF%87%E9%98%B2%E7%81%AB%E5%A2%99%E7%9A%84%E6%B5%81%E7%A8%8B.png)

filter表：负责过滤功能，防火墙；内核模块：iptables_filter

nat表：network address translation，网络地址转换功能；内核模块：iptable_nat

mangle表：拆解报文，做出修改，并重新封装 的功能；iptable_mangle

raw表：关闭nat表上启用的连接追踪机制；iptable_raw

---

实际使用：通过”表”作为操作入口，对规则进行定义的

（高级功能，如：网址过滤。）raw     表中的规则可以被哪些链使用：PREROUTING，OUTPUT

（数据包修改（QOS），用于实现服务质量。）mangle  表中的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING

（地址转换，用于网关路由器。）nat     表中的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（centos7中还有INPUT，centos6中没有）

（包过滤，用于防火墙规则。）filter  表中的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT



iptables -t 表名 <-A/I/D/R> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP/源子网> --sport 源端口 <-d 目标IP/目标子网> --dport 目标端口 -j 动作

## 2、ipvs
#### 2.1、service和kube-proxy的关系
service 的代理是 kube-proxy
kube-proxy 运行在所有节点上，它监听 apiserver 中 service 和 endpoint 的变化情况，创建路由规则以提供服务 IP 和负载均衡功能。简单理解此进程是Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上，而kube-proxy底层又是通过iptables和ipvs实现的。

#### 2.2 iptables原理
Kubernetes从1.2版本开始，将iptables作为kube-proxy的默认模式。iptables模式下的kube-proxy不再起到Proxy的作用，其核心功能：通过API Server的Watch接口实时跟踪Service与Endpoint的变更信息，并更新对应的iptables规则，Client的请求流量则通过iptables的NAT机制“直接路由”到目标Pod。

#### 2.3 ipvs原理
IPVS在Kubernetes1.11中升级为GA稳定版。IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张，因此被kube-proxy采纳为最新模式。

在IPVS模式下，使用iptables的扩展ipset，而不是直接调用iptables来生成规则链。iptables规则链是一个线性的数据结构，ipset则引入了带索引的数据结构，因此当规则很多时，也可以很高效地查找和匹配。

可以将ipset简单理解为一个IP（段）的集合，这个集合的内容可以是IP地址、IP网段、端口等，iptables可以直接添加规则对这个“可变的集合”进行操作，这样做的好处在于可以大大减少iptables规则的数量，从而减少性能损耗。

#### 2.4 kube-proxy ipvs和iptables的异同
iptables与IPVS都是基于Netfilter实现的，但因为定位不同，二者有着本质的差别：iptables是为防火墙而设计的；IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张。

#### 2.5 kube-proxy ipvs和iptables的异同
与iptables相比，IPVS拥有以下明显优势：

- 为大型集群提供了更好的可扩展性和性能；
- 支持比iptables更复杂的复制均衡算法（最小负载、最少连接、加权等）；
- 支持服务器健康检查和连接重试等功能；
- 可以动态修改ipset的集合，即使iptables的规则正在使用这个集合。
#### 2.6 kube-proxy工作模式
##### 2.6.1 userspace
 userspace是在用户空间，通过kube-proxy来实现service的代理服务
##### 2.6.1 iptables
该模式完全利用内核iptables来实现service的代理和LB, 这是K8s在v1.2及之后版本默认模式
##### 2.6.1 ipvs
  在kubernetes 1.8以上的版本中， ipvs 是基于 NAT 实现的，通过ipvs的NAT模式，对访问k8s service的请求进行虚IP到POD IP的转发。
  当创建一个 service 后，kubernetes 会在每个节点上创建一个网卡，同时帮你将 Service IP(VIP) 绑定上，此时相当于每个 Node 都是一个 ds，而其他任何 Node 上的 Pod，甚至是宿主机服务(比如 kube-apiserver 的 6443)都可能成为 rs；
## 3、bps


参考：

https://www.zsythink.net/archives/1199 （iptables–入门版）

https://blog.csdn.net/u011537073/article/details/82685586 （iptables-详细参数说明）

https://www.tigera.io/blog/introducing-the-calico-ebpf-dataplane/ （Introducing the Calico eBPF dataplane）

https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables （Why is the kernel community replacing iptables with BPF?）

http://www.dockone.io/article/10484 （腾讯TKE用eBPF）

https://cilium.io/blog/2021/05/11/cni-benchmark （CNI Benchmark: Understanding Cilium Network Performance）

https://davidlovezoe.club/wordpress/archives/1122 （LINUX超能力BPF技术介绍及学习分享）

https://www.cnblogs.com/zjz20/p/13452717.html （iptables和ipvs）
