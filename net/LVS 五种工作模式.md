```
cip：Client IP，客户端地址
vip：Virtual IP，虚IP
rip：Real IP，后端RS地址
RS: Real Server 后端真正提供服务的机器
LB： Load Balance 负载均衡器
LVS： Linux Virtual Server
sip： source ip，源IP
dip： destination ip，目的IP
NAT： Network Address Translation，网络地址转换
SNAT: Source Network Address Translation，源地址转换
DNAT: Destination Network Address Translation，目的地址转换
```


**一、DR模式**

1. 请求流量(sip 200.200.200.2, dip 200.200.200.1) 先到达 LVS
2. 然后LVS，根据负载策略挑选众多 RS中的一个，然后将这个网络包的MAC地址修改成这个选中的RS的MAC
3. LVS将处理过后的数据包丢给交换机，交换机根据二层MAC信息将这个包丢给选中的RS
4. 接收到数据包的RS看到MAC地址是自己的、dip也是自己的（VIP配置在lo），愉快地收下并处理，并根据路由表将回复包（sip 200.200.200.1， dip 200.200.200.2）返回给交换机
5. 回复包（sip 200.200.200.1， dip 200.200.200.2）经过交换机直接回复给client（不再走LVS）

<br/>

**二、NAT模型**

<br/>

1. client发出请求（sip 200.200.200.2，dip 200.200.200.1）
2. 请求包到达lvs，lvs修改请求包为（sip 200.200.200.2， dip 10.10.10.2）
3. 请求包到达rs， rs回复（sip 10.10.10.2，dip 200.200.200.2）
4. 这个回复包不能直接给client，因为sip不是200.200.200.1（vip）会被reset掉
5. 设置lvs为网关，所以这个回复包先走到lvs，lvs有机会修改sip
6. lvs修改sip为VIP，修改后的回复包（sip 200.200.200.1，dip 200.200.200.2）发给client

<br/>

**三、full NAT 模型**

<br/>

1. client发出请求（sip 200.200.200.2 dip 200.200.200.1）
2. 请求包到达lvs，lvs修改请求包为（sip 200.200.200.1， dip rip） 注意这里sip/dip都被修改了
3. 请求包到达rs， rs回复（sip rip，dip 200.200.200.1）
4. 这个回复包的目的IP是VIP(不像NAT中是 cip)，所以LVS和RS不在一个vlan通过IP路由也能到达lvs
5. lvs修改sip为vip， dip为cip，修改后的回复包（sip 200.200.200.1，dip 200.200.200.2）发给client

**四、IP TUN模型**

1. 请求包到达LVS后，LVS将请求包封装成一个新的IP报文
2. 新的IP包的目的IP是某一RS的IP，然后转发给RS
3. RS收到报文后IPIP内核模块解封装，取出用户的请求报文
4. 发现目的IP是VIP，而自己的tunl0网卡上配置了这个IP，从而愉快地处理请求并将结果直接发送给客户

***

参考：

https://segmentfault.com/a/1190000019907036

