# OCI

 OCI（Open Container Initiative）即开放的容器运行时规范，目的在于定义一个容器运行时及镜像的相关标准和规范，其中包括

- runtime-spec：容器的生命周期管理
- image-spec：镜像的生命周期管理，

实现OCI标准的容器运行时有**runc，kata**等。

# runc

runc(run container)是一个基于OCI标准实现的一个轻量级容器运行工具，**用来创建和运行容器。**

Containerd是用来维持通过runc创建的容器的运行状态

即runc用来创建和运行容器，containerd作为常驻进程用来管理容器。

```sh
[caas_dev@sz02 ~]$ runc
NAME:
   runc - Open Container Initiative runtime

runc is a command line client for running applications packaged according to
the Open Container Initiative (OCI) format and is a compliant implementation of the
Open Container Initiative specification.
# runc 是一个命令行客户端，用于运行根据以下格式打包的应用程序：
# 开放容器倡议 (OCI) 格式，并且是开放容器倡议规范的合规实现。


Containers are configured using bundles. A bundle for a container is a directory
that includes a specification file named "config.json" and a root filesystem.
The root filesystem contains the contents of the container.
```

<br/>

# Containerd

containerd（container daemon）是一个daemon进程用来管理和运行容器，可以用来拉取/推送镜像和管理容器的存储和网络。其中可以调用runc来创建和运行容器。

<br/>

# docker与containerd、runc的关系图

Docker CLI (你敲的命令) -> Docker Daemon (后台服务) -> containerd (容器调度管理) -> runc (实际干活的)

<br/>

![](https://k8s.huweihuang.com/project/~gitbook/image?url=https%3A%2F%2Fres.cloudinary.com%2Fdqxtn0ick%2Fimage%2Fupload%2Fv1631847625%2Farticle%2Fkubernetes%2Fcontainerd%2Fcontainer-ecosystem-docker.drawio.png&width=768&dpr=4&quality=100&sign=912c00a1&sv=2)

Docker/containerd 负责指挥：比如下载镜像、准备网络、管理存储卷。

runc 负责执行：它拿到指令和文件系统后，直接调用 Linux 内核的特性（Namespace, Cgroups 等）来启动一个隔离的进程（即容器）。

<br/>

# runc 的弊端与局限性

## 安全性的“先天不足”：共享内核

runc 最大的局限性源于它与宿主机共享同一个操作系统内核。

逃逸风险（Container Escape）： 如果攻击者在容器内利用了 Linux 内核漏洞，他们可以直接攻击宿主机内核，从而突破容器边界。

权限攻击： runc 历史上多次爆发高危漏洞（如 CVE-2019-5736, CVE-2024-21626, CVE-2025-31133）。由于 runc 本身是以 root 权限运行的（除非使用 Rootless 模式），攻击者可以通过符号链接（Symlink）、/proc 文件系统重定向或挂载竞态条件（Race Conditions）来获取宿主机的 Root 权限。

内核破坏： 容器内的进程可以直接向内核发起大量的系统调用（Syscalls）。虽然有 seccomp 过滤，但内核攻击面依然巨大，一个恶意容器可能导致整个系统崩溃（Panic）。

## 资源隔离的“软局限”

runc 依赖 Linux 的 Cgroups 和 Namespaces。这种隔离属于“软隔离”，存在以下问题：

资源视图不一致： 在 runc 容器内运行 top 或查看 /proc/meminfo，默认会看到宿主机的全部内存和 CPU 信息。虽然内核限制了实际用量，但很多应用程序（如 Java JVM、数据库）会根据看到的硬件信息来预分配内存，导致在容器内频繁触发 OOM（内存溢出）。

无法隔离所有内核资源： 并非所有内核子系统都实现了 Namespace 隔离。例如，内核时间、某些内核日志、以及某些复杂的驱动程序接口是全系统共享的。一个容器内的恶意操作（如频繁修改系统时间或耗尽内核 entropy）可能影响全局。
