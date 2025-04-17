# 1、架构拓扑

![](https://i0.hdslb.com/bfs/article/029dae5859ef716b2bc16099c4ad1dbd368973997.png)

![](https://ask.qcloudimg.com/http-save/yehe-6638172/x2crhv7hyj.png)

- Bird：BGP客户端，Calico在每个节点上的都会部署一个BGP客户端，它的作用是将Felix的路由信息读入内核，并通过BGP协议在集群中分发。当Felix将路由插入到Linux内核FIB中时，BGP客户端将获取这些路由并将它们分发到部署中的其他节点。这可以确保在部署时有效地路由流量。
- BGP Router Reflector：大型网络仅仅使用 BGP client 形成 mesh 全网互联的方案就会导致规模限制，所有节点需要 N^2 个连接，为了解决这个规模问题，可以采用 BGP 的 Router Reflector 的方法，使所有 BGP Client 仅与特定 RR 节点互联并做路由同步，从而大大减少连接数。
- Felix：calico的核心组件，运行在每个节点上。主要的功能有接口管理、路由规则、ACL规则和状态报告
- Etcd：保证数据一致性的数据库，存储集群中节点的所有路由信息。为保证数据的可靠和容错建议至少三个以上etcd节点。
- Orchestrator plugin：协调器插件负责允许kubernetes或OpenStack等原生云平台方便管理Calico，可以通过各自的API来配置Calico网络实现无缝集成。如kubernetes的cni网络插件。

<br/>

## 1.1 两种模式

- 路由反射模式Router Reflection（RR）
  - RR模式 中会指定一个或多个BGP Speaker为RouterReflection，它与网络中其他Speaker建立连接，每个Speaker只要与Router Reflection建立BGP就可以获得全网的路由信息。在calico中可以通过Global Peer实现RR模式。
- 全互联模式(node-to-node mesh)
  - 全互联模式 每一个BGP Speaker都需要和其他BGP Speaker建立BGP连接，这样BGP连接总数就是N^2，如果数量过大会消耗大量连接。如果集群数量超过100台官方不建议使用此种模式。

## 1.2 Pod 1 访问 Pod 2

![](https://ask.qcloudimg.com/http-save/yehe-6638172/j41ngbry2b.png)

1. 数据包从 Pod1 出到达Veth Pair另一端（宿主机上，以cali前缀开头）
2. 宿主机根据路由规则，将数据包转发给下一跳（网关）
3. 到达 Node2，根据路由规则将数据包转发给 cali 设备，从而到达 Pod2。

其中，这里最核心的 **下一跳 **路由规则，就是由 Calico 的 Felix 进程负责维护的。这些路由规则信息，则是通过 BGP Client 中 BIRD 组件，使用 BGP 协议来传输。

不难发现，Calico 项目实际上将集群里的所有节点，都当作是边界路由器来处理，它们一起组成了一个全连通的网络，互相之间通过 BGP 协议交换路由规则。**这些节点，我们称为 BGP Peer。**

<br/>

## 1.2 IPAM

```yaml
apiVersion：crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: ippool-test-0
spec:
  blockSize: 32
  cidr: 1.1.1.0/24
  ipipMode: Never
  natOutgoing: false
  nodeSelector: “!all()”
  vxlanMode: Never
```

- nodeSelector: 该字段与Kubernetes节点的Label进行映射。默认为all(),表示所有节点均可使用。设置为!all()，表示所有node均不可自动使用，可通过设置命名空间或者POD的注解，实现IPPool的绑定。
- block/blockSzie: block主要功能是路由聚合，减少对外宣告路由条目。block在POD所在节点自动创建，如在worker01节点创建1.1.1.1的POD时，blocksize为29，则该节点自动创建1.1.1.0/29的block，对外宣告1.1.1.0/29的BGP路由，并且节点下发1.1.1.0/29的黑洞路由和1.1.1.1/32的明细路由。在IBGP模式下，黑洞路由可避免环路。如果blockSize设置为32，则不下发黑洞路由也不会造成环路，缺点是路由没有聚合，路由表项会比较多，需要考虑交换机路由器的容量。
- Calico创建block时，会出现借用IP的情况。如在 worker01节点存在1.1.1.0/29的block，由于worker01节点负载很高，地址为1.1.1.2的POD被调度到worker02节点，这种现象为IP借用。woker02节点会对外宣告1.1.1.2/32的明细路由，在IBGP模式下，交换机需要开启RR模式，将路由反射给worker01上，否则在不同worker节点的同一个block的POD，由于黑洞路由的存在，导致POD之间网络不通。可通过ipamconfigs来管理是否允许借用IP(strictAffinity)、每个节点上最多允许创建block的数量(maxBlocksPerHost)等。
- Block（子网块）
  - Calico 将 IPPool 按 blockSize 拆分为多个较小的子网块（如 /26）。
  - 每个node分配一个 Block，节点上的所有 Pod 均从该 Block 中分配 IP。
示例：IPPool 192.168.0.0/16 + blockSize: 26 → 每个 Block 为 /26（如 192.168.1.0/26）。
- 选择合理的 Block 大小
  - 公式：blockSize = 32 - log2(最大 Pod 数/节点 + 2)
  - 若节点最多运行 30 个 Pod：
    - 需要至少 30 + 2 = 32 个 IP → blockSize = 26（2^(32-26) = 64，满足需求）。

## 1.3 rr路由学习

<br/>

<br/>

---

https://cloud.tencent.com/developer/article/1638845

https://blog.csdn.net/margu_168/article/details/132045839
