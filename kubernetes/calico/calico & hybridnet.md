||calico BGP|hybridnet BGP|hybridnet global BGP|
|--|--|--|--|
|ip资源CRD|ippool|Network|Network|
|rr广播CRD|block|subnet （NO_EXPORT）|明细路由 （不带NO_EXPORT）|
|是否支持自动创建子网断资源|block可在ippool资源充足时自动创建。ippool需手动创建。|subnet消耗完需手动创建||

hybridnet BGP VS global BGP ：

前者在进行rr 传播的时候，是以subnet聚合后传播，且只在当前AS内（因为有noexport）。

后者 是以 明细路由出传播，且 可以跨AS传播（因为没有noexport ）

<br/>

```yaml
### calico
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: my-ipv4-pool  # IP 池名称
spec:
  cidr: 10.244.0.0/16  # IP 地址范围
  blockSize: 26        # 每个 Block 的子网大小（默认 26，即每个 Block 包含 64 个 IP）
  ipipMode: Never      # 是否启用 IP-in-IP 隧道（可选 Always/CrossSubnet/Never）
  natOutgoing: true    # 是否对出向流量做 SNAT
  nodeSelector: all()  # 节点选择器（默认应用于所有节点）
  disabled: false      # 是否禁用此 IPPool、
  
---
apiVersion: projectcalico.org/v3
kind: Block
metadata:
  cidr: 10.244.5.0/26  # Block 的 CIDR 范围
spec:
  affinity: node:node01  # 绑定到特定节点（如 node01）
  allocations:
    - 0: 10.244.5.1      # 已分配的 IP 地址（例如索引 0 对应 IP 10.244.5.1）
  unallocated:
    - 1-63               # 未分配的 IP 索引范围
---
apiVersion: v1
kind: Pod
metadata:
  name: calico-pod
  annotations:
    # 指定 Pod 从特定 IPPool 分配 IP（需提前创建该 IPPool）
    cni.projectcalico.org/ipv4pools: '["my-ipv4-pool"]'
    # 可选：固定 Pod 的 IP 地址（需确保 IP 未被占用）
    # cni.projectcalico.org/ipv4address: 10.244.5.100/24
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  # 可选：通过节点选择器调度到特定节点
  nodeSelector:
    kubernetes.io/hostname: node01
```

```yaml
# aliyun hybridnet
apiVersion: networking.alibaba.com/v1
kind: Network
metadata:
  name: underlay-network  # 网络名称
spec:
  netID: 100              # 网络唯一标识（同一集群内不可重复）
  type: Underlay          # 网络类型（Underlay/Overlay）
  accessMode: Bridge      # 网络接入模式（Bridge/SR-IOV）
  default: false          # 是否作为默认网络（若为 true，未指定网络的 Pod 会分配到此网络）
---
apiVersion: networking.alibaba.com/v1
kind: Subnet
metadata:
  name: underlay-subnet  # 子网名称
spec:
  network: underlay-network  # 所属 Network 名称
  range:                   # IP 地址范围（CIDR 格式）
    cidr: 192.168.1.0/24
    gateway: 192.168.1.1   # 网关地址（Underlay 网络必须指定）
  reservedIPRanges:        # 保留 IP 段（不自动分配）
    - 192.168.1.1-192.168.1.50
  nodeSelector:            # 子网可分配的节点标签选择器
    network: underlay      # 仅标签为 network=underlay 的节点可使用此子网
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    networking.alibaba.com/network: underlay-network  # 指定所属 Network
spec:
  containers:
  - name: nginx
    image: nginx
```
