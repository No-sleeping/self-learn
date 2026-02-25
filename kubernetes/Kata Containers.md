![](https://katacontainers.io/static/589e3d905652847b22c395fe6bbbace7/663f4/katacontainers_architecture_diagram.jpg)

<br/>

# 一、组件介绍

## 1.1 kata shim V2

这是 Kata 2.x 引入的最重要的组件，它实现了 Containerd 的 Runtime v2 (Shim API)。

作用：

- 生命周期管理： 它负责启动和停止底层的 Hypervisor（如 QEMU 或 Firecracker）。
- API 翻译： 它接收来自 Containerd 的 OCI 指令（如 Create, Start, Kill），并将这些指令转换为通过 VSock 发送给 Kata Agent 的 gRPC 请求。
- IO 流处理： 它负责处理容器的标准输入/输出（stdin/stdout/stderr）。它读取 VM 传出来的流，转发给 Containerd。
- 取代 Proxy： 在旧版 Kata 中，需要 kata-proxy 来处理多路复用，但在 Shim v2 中，Shim 进程直接处理所有 IO 和信号，架构大大简化。

部署形态： 对于每一个 Pod，宿主机上会运行一个 containerd-shim-kata-v2 进程。

## 1.2 Kata Agent

这是运行在 Guest VM 内部的守护进程，通常是 VM 启动后的第一个进程（PID 1）。

作用：

- 指令执行： 它在 VM 内部监听 gRPC 服务。当 Shim 发来“启动容器”的指令时，Agent 会在 VM 内执行实际的操作。
- 容器管理： Agent 使用 libcontainer（这也是 runc 的核心库）在 VM 内部创建 Namespace 和 Cgroups，真正拉起用户的应用进程。
- 生命周期监控： 监控容器进程的状态，如果进程退出，Agent 会将退出码返回给宿主机的 Shim。
- 热插拔处理： 协助处理热插拔 CPU、内存或设备的操作。

## 1.3 VSock 

VSock 是 Linux 内核提供的一种专门用于 宿主机与虚拟机之间 高效通信的 Socket 机制。

作用：

- 控制通道： Kata Shim（宿主机）和 Kata Agent（虚拟机）之间所有的控制指令（gRPC）都通过 VSock 传输。
- 数据通道： 容器的日志输出、终端交互流也通过 VSock 传输。
- 为什么不用 TCP/IP？
VSock 不需要配置复杂的虚拟网卡和 IP 地址。
VSock 仅在本地内存中复制数据，比经过完整的 TCP/IP 协议栈要快得多，延迟更低，且更安全（无法被外部网络访问）。

## 1.4 Hypervisor

Kata 支持多种 Hypervisor，由 Shim 负责配置和拉起。

- QEMU: 默认选项，兼容性最强，支持各种设备直通（Device Passthrough，如 GPU）。
- Firecracker: AWS 开源的微型虚拟机管理程序，启动速度极快，但功能有限（不支持文件系统共享，需配合 block device）。
- Cloud Hypervisor (CLH): 基于 Rust，专注于云原生负载，是未来的趋势之一。

## 1.5 Virtio-fs 

这是解决“容器镜像在宿主机，但要在虚拟机里跑”这个问题的关键技术。

- 作用： 它允许虚拟机像访问本地文件系统一样，直接访问宿主机上的目录。
- 流程： Containerd 把镜像解压在宿主机 -> Kata Shim 通过 Virtio-fs 把这个目录挂载给 VM -> VM 内的 Agent 看到目录，以此为根文件系统启动容器。
- 优势： 避免了把整个镜像拷贝进虚拟机的巨大开销，实现了秒级启动。
