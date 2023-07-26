# bridge

当在操作系统中安装了docker之后，docker会自动生成一个名为“docker0”的虚拟网桥。

大家可以把这个网桥想象成一台虚拟交换机，这台交换机可以工作在而二层也可以工作在三层，默认工作在第二层；不过一旦你给这个虚拟网桥分配了IP地址，它就工作在了三层。

而**每当生成一个docker容器，那么就会有一条虚拟网线连接了docker容器的eth0端口和”docker0”这台交换机一个名为vethxx的端口**，那么这也就非常直观地意味着所有连接在这个”docker0”交换机上的docker容器都可以互相通信，到这一步，docker容器以及可以访问外部网络了，但是我们还无法从外部访问docker容器。如果在这个过程中，你使用了-p命令，那么就是做了一次端口转发，实现了我们从外部访问docker容器，网络拓扑图如下：

![](https://k.sinaimg.cn/n/sinakd20230227s/142/w600h342/20230227/322f-5c69066a92b980242a0fdcc324385d54.jpg/w700d1q75cms.jpg)

<br/>

参考：

https://t.cj.sina.com.cn/articles/view/1823348853/6cae1875020018aby

# macvlan 

总结：macvlan 是一种**网卡虚拟化技术**；macvlan 的四种通信模式，常用模式是 **bridge**。

- Private 模式
  
  禁止构建在同一物理接口上的多个MAC VLAN实例（容器接口）彼此间的通信，即便外部的物理交换机支持“发夹模式”也不行。
- Bridge 模式
- VEPA 模式
  
  允许构建在同一物理接口上的多个MAC VLAN实例（容器接口）彼此间的通信，但需要外部交换机启用发夹模式，或者存在报文转发功能的路由器设备。
- Passthru 模式

![](https://upload-images.jianshu.io/upload_images/26746788-f8ddd3f385970b68.png?imageMogr2/auto-orient/strip|imageView2/2/w/437/format/webp)

参考：

https://www.cnblogs.com/bakari/p/10893589.html

https://www.jianshu.com/p/9b8c370baca1

# ipvlan

![](https://hicu.be/wp-content/uploads/2016/03/linux-ipvlan.png)

IPVlan 有两种模式，一种 L2 模式，一种 L3 模式。同一张父网卡同一时间只能使用一种模式。

- L2
  
  这种模式很像是网桥，可以简单理解成把虚拟化出来的子网卡当成插在父网卡这个 “网桥” 上的网卡设备，然后对于**同一个网段之间的网卡通信**，父网卡的作用类似于 “网桥”。
  
  ![](https://hicu.be/wp-content/uploads/2016/03/linux-ipvlan-l2-mode.png)
- L3
  
  这种模式比较像是一个路由器，它允许创建出来的子网卡**不在同一个网段内**。
  
  ![](https://hicu.be/wp-content/uploads/2016/03/linux-ipvlan-l3-mode-1.png)

<br/>

<br/>

参考：

https://zhuanlan.zhihu.com/p/573914523

https://hicu.be/macvlan-vs-ipvlan

# SR-IOV

#  underlay

Underlay网络就是传统IT基础设施网络，由交换机和路由器等设备组成，借助以太网协议、路由协议和VLAN协议等驱动，它还是Overlay网络的底层网络，为Overlay网络提供数据通信服务。容器网络中的Underlay网络是指借助驱动程序将宿主机的底层网络接口直接暴露给容器使用的一种网络构建技术，较为常见的解决方案有**MAC VLAN、IP VLAN和直接路由**等。

<br/>

# overlay

<br/>

https://blog.csdn.net/qq_39965059/article/details/126022945  （【K8S】详解容器网络中的overlay、underlay）

https://www.cnblogs.com/cyh00001/p/16646062.html  （overlay与underlay通信总结 ）

# Calico

- Calico overlay 模式：Calico IPIP或VXLAN模式
- Calico underlay 模式：calico BGP模式，

https://blog.csdn.net/u011619480/article/details/128613875

# BGP

<br/>

<br/>

https://blog.csdn.net/u011619480/article/details/128613875  （Calico）
