## 预备知识

1. veth pair
veth 从名字上来看是 Virtual ETHernet 的缩写.
把从一个 network namespace 发出的数据包转发到另一个 namespace
veth 设备是成对的，一个是 container 之中，另一个在 container 之外，即在真实机器上能看到的。
2. tunl0
3. CNI
   
   CNI用于连接容器管理系统和网络插件。
   
   提供一个容器所在的network namespace，将network interface插入该network namespace中（比如veth的一端），并且在宿主机做一些必要的配置（例如将veth的另一端加入bridge中），最后对namespace中的interface进行IP和路由的配置。

<br/>

## 【IPIP】

![](https://s8.51cto.com/images/20200518/1589770646867489.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jZG0ueXAxNC5jbi9pbWcvY2FsaWNvLWlwaXAtMS5wbmc?x-oss-process=image/format,png)

![](https://img2018.cnblogs.com/blog/1060878/201904/1060878-20190414173605263-65569449.png)

<br/>

### 【ipip】不同node下 pod ping pod 网络怎么走

从pod2（192.168.166.145 ）ping pod1（192.168.104.20）

1、pod-2中的eth0(即图中的vthe0)与Cali.4a是一对veth pair，因此，Cali.4a接收到的ip流向一定与vthe0相同，为 192.168.166.145>192.168.104.20。

2、经过tunl0的ip报会被再封上一层ip。通过node1的route规则，会发往ens33，因此我们在ens33处的抓包结果为 192.168.10.11 > 192.168.10.12: IP 192.168.166.145>192.168.104.20

3、4其实就是1、2的逆过程，检查node2的route表即可知道流向。ens33将ipip拆封后，将流量发给tunl0，tunl0再转发给cali.90。

![](https://s7.51cto.com/images/20200518/1589771015195826.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 【ipip】同node下 pod ping pod 网络怎么走

如果是同一个node内的两个pod进行访问，通过上节的route规则就可以知道，Calico会为每一个node分配一小段网络，同时会为每个pod创建一个“入”的ip route规则。

如下图所示，当从pod1访问pod3时，Cali.90是直接发出192.168.104.20-> 192.168.104.21流量的，在node2的ip route中，发往192.168.104.21的ip报直接会被转发到cali.2f，不会用到tunl0，只有在node间访问的时候才会使用tunl0进行ipip封装！

![](https://s9.51cto.com/images/20200518/1589771227604559.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

<br/>

## 【BGP】

1、数据包从 Pod1 出到达Veth Pair另一端（宿主机上，以cali前缀开头）

2、宿主机根据路由规则，将数据包转发给下一跳（网关）

3、到达 Node2，根据路由规则将数据包转发给 cali 设备，从而到达 Pod2。

![](https://img2018.cnblogs.com/blog/1060878/201904/1060878-20190415165320714-135136611.png)

<br/>

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9jZG0ueXAxNC5jbi9pbWcvY2FsaWNvLWJncC0xLnBuZw?x-oss-process=image/format,png)

![](https://blog.csdn.net/qq_23435961/article/details/106660196)

***

参考：

https://blog.csdn.net/sld880311/article/details/77650937（veth pair）

https://www.cnblogs.com/bakari/p/10613710.html

https://blog.51cto.com/u_14268033/2496122 （IPIP模式）

https://www.cnblogs.com/goldsunshine/p/10701242.html（calio）

https://blog.csdn.net/qq_23435961/article/details/106660196  （calio）

https://cloud.tencent.com/developer/article/old/1922673  （BGP基本概念）
