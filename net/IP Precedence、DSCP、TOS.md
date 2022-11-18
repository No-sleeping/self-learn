# IP Precedence

![](https://img-blog.csdnimg.cn/73d445e26f4449caac61da96ff39b637.png)

![](https://img-blog.csdnimg.cn/20210414152437762.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhbmRhb3l1,size_16,color_FFFFFF,t_70)

IPv4中有8bit作为TOS字段，一开始RFC791定义了TOS前三位为IP Precedence，划分了8个优先级，可用于流分类，数值越大表示优先级越高。

7 预留（Reserved）
6 预留（Reserved）
5 语音（Voice）
4 视频会议（Video Conference）
3 呼叫信号（Call Signaling）
2 高优先级数据（High-priority Data）
1 中优先级数据（Medium-priority Data）
0 尽力服务数据（Best-effort Data）

# DSCP

随着网络的发展，8个优先级已经不能满足实际需要，于是RFC2474又对TOS重新进行了定义，把前六位定义为DSCP差分服务代码（Differentiated Services Code Point），后两位保留。

由于DSCP和IP PRECEDENCE是**共存**的，于是存在了一些兼容性的问题，DSCP的可读性比较差，比如DSCP 43我们并不知道对应着IP PRECEDENCE的什么取值，于是就把DSCP进行了进一步的分类。DSCP总共分成了4类：

<br/>

1. Default(BE) 000 000： 默认值
2. Class Selector(CS) xxx 000 ：CS的DSCP后三位为0，也就是说CS仍然沿用了IP Precedence只不过CS定义的DSCP=IP Precedence*8，比如：CS6（110 000）=6 x 8=48，CS7（111 000）=7 x 8=56    **（8，16，24，32，40，48，56）**
   
   ![](https://img-blog.csdnimg.cn/20210406204014602.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xlZ2VuZDA1MDcwOQ==,size_16,color_FFFFFF,t_70)
   
   <br/>
3. Expedited Forwarding(EF) 101 110 ：EF含义为加速转发，也可以看作为IP Precedence为5，是一个比较高的优先级，取值为101110(46)，但是RFC并没有定义为什么EF的取值为46。**推荐值为46（101110）**
4. Assured Forwarding(AF) aaa bb0：AF分为两部分，a部分（IP优先级）和b部分 如下图：
   
   a部分为3 bit仍然可以和IP Precedence对应；

b部分为2 bit表示丢弃性，可以表示3个丢弃优先级，可以应用于RED或者WRED。

   目前a部分有三个bit最大取值为8，但是目前只用到了1~4。为了迅速的和10进制转换，可以用如下方法，先把10进制数值除8得到的整数就是AF值，余数换算成二进制看前两位就是丢弃优先级，比如34/8=4余数为2，2 换算成二进制为010，那么换算以后可以知道34代表AF4丢弃优先级为middle的数据报。
确定转发（AF），定义了4个服务等级，每个服务等级有3个下降过程，因此使用了12个DSCP值（（10，12，14），（18，20，22），（26，28，30），（34，36，38））

   ![](https://img-blog.csdnimg.cn/ce7d16415c7540b18cf9e27960f0d1e8.png)

   ## DSCP和TOS对照

![](https://img-blog.csdnimg.cn/f5fc3ad5df764a6e9223bdab6ebe8b34.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5YuJ5peP,size_20,color_FFFFFF,t_70,g_se,x_16)

<br/>

##  实践

<br/>

```sh
#client：  
ping -Q 40  $server_ip

#server：
sudo tcpdump -v -i bond0.104 host $client_ip
tcpdump: listening on bond0.104, link-type EN10MB (Ethernet), capture size 262144 bytes
14:38:42.454606 IP (tos 0x28, ttl 63, id 0, offset 0, flags [DF], proto ICMP (1), length 84)
    $client_ip > $server_ip: ICMP echo request, id 57393, seq 1, length 64
14:38:42.454666 IP (tos 0x28, ttl 64, id 33187, offset 0, flags [none], proto ICMP (1), length 84)
    $server_ip > $client_ip: ICMP echo reply, id 57393, seq 1, length 64
    
    
    
    
#client：  
ping -Q 64  $server_ip

#server：
14:39:34.907448 IP (tos 0x40, ttl 63, id 0, offset 0, flags [DF], proto ICMP (1), length 84)
    $client_ip > $server_ip: ICMP echo request, id 60210, seq 1, length 64
14:39:34.907517 IP (tos 0x40, ttl 64, id 6191, offset 0, flags [none], proto ICMP (1), length 84)
    $server_ip > $client_ip: ICMP echo reply, id 60210, seq 1, length 64

```

---

参考 ： 

https://blog.csdn.net/legend050709/article/details/115470228

https://blog.csdn.net/qq_33681684/article/details/123747965
