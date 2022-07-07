## 一、tcp、tls关系

> ***一个完整的https流：
应该是先TCP三次握手之后，再TLS的四次握手。***

![](https://st.imququ.com/i/webp/static/uploads/2015/11/tls-handshake.png.webp)

## 二、2.1 tls概念

![](https://img-blog.csdnimg.cn/2020070713434159.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA0NTMyOA==,size_16,color_FFFFFF,t_70)

 SSL（Secure Socket Layer，安全套接字层）协议是位于OSI七层模型中的表示层（在五层模型中属于应用层）的可靠的面向连接的协议，SSL通过互相认证、使用摘要算法确保完整性和使用加密算法确保私密性，使客户端和服务器之间实现了安全的通讯。

SSL是基于HTTP之下TCP之上的一个协议层，是基于HTTP标准并对TCP传输数据时进行加密，所以HPPTS是HTTP+SSL/TCP的简称。

在SSL更新到3.0时，IETF对SSL3.0进行了标准化，并添加了少数机制(但是几乎和SSL3.0无差异)，标准化后的IETF更名为TLS1.0(Transport Layer Security 安全传输层协议)，可以说TLS就是SSL的新版本。

<br/>

###  2.2 tls握手详解

TLS四次握手简图：

![](https://img-blog.csdnimg.cn/20210516162401296.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0dhb3ppaGFuZzc3Nw==,size_16,color_FFFFFF,t_70#pic_center)

#### step1 : Client Hello

在这一步，客户端主要向服务器发送以下信息：

1. 客户端支持的 SSL/TLS 协议版本，如 TLS 1.2 版本。
2. 客户端生产的随机数（Client Random），后面用于生产「会话秘钥」。
3. 客户端支持的密码套件列表，如 RSA 加密算法。
4. 服务端的数字证书。

<br/>

#### step2: Server Hello，Chane Cipher spec，Encrypted Handshake Message

服务器收到客户端请求后，向客户端发出响应，也就是 SeverHello。服务器回应的内容有如下内容：

1. 确认 SSL/ TLS 协议版本，如果浏览器不支持，则关闭加密通信。
2. 服务器生产的随机数（Server Random），后面用于生产「会话秘钥」。
3. 确认的密码套件列表，如 RSA 加密算法。
4. 服务器的数字证书。

<br/>

#### step3 :客户端回应

3.1 验证证书有效（是否已经过期、证书中的域名是否与实际域名一致、证书是否由可信机构颁发）

3.2 如果任何一个环境出现了问题，浏览器会告警。如果全部通过，浏览器会生成一串新的随机数（Premaster secret ），并用证书中提供的公钥加密。

此时，浏览器会根据前三次握手中的三个随机数：

- Client random
- Server random
- Premaster secret
  
  通过一定的算法来生成 “会话密钥” （Session Key），这个会话密钥就是接下来双方进行对称加密解密使用的密钥！

3.3 Client Key Exchange（服务器端有一个密钥对）

客户端用服务端的公钥将PreMaster Sercet发送给服务器，服务器则会用自己的私钥解密得出PreMaster 。

到这里客户端和服务器都拥有了三个随机数R1、R2和Pre-master，两边再用相同的算法和这三个随机数生成一个密钥，用于握手结束后传输数据的对称加密。

<br/>

#### step4: 服务端回应

服务器收到客户端的第三个随机数（ Premaster secret） 之后，使用同样的算法计算出 “会话密钥” （Session Key）。

1. 加密通信算法改变通知，表示随后的信息都将用「会话秘钥」加密通信。
2. 服务器握手结束通知，表示服务器的握手阶段已经结束。这一项同时把之前所有内容的发生的数据做个摘要，用来供客户端校验。

<br/>

***

## 三、SSL/TLS协议运作流程明细

![](https://img-blog.csdnimg.cn/20210129121236557.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzUwMDg0NzE4,size_16,color_FFFFFF,t_70#pic_center)

![](https://raw.githubusercontent.com/No-sleeping/self-learn/main/images/net/SSL%E6%8F%A1%E6%89%8B%20(3).png)

<br/>

o Client Hello：客户端向服务端打招呼；携带支持的协议、支持的安全套件供服务端选择；

o Server Hello：服务端回应客户客户端的招呼信息；结合客户端的信息，选择合适的加密套件；

o Certificate：服务端向客户端发送自己的数字证书（此证书包含服务端的公钥），以实现验证身份；

o Server Key Exchange：服务端向客户端发送基于选择的加密套件生成的公钥（此公钥为椭圆曲线的公钥，用于协商出对称加密的密钥）；

o Server Hello Done：服务端向客户端表示响应结束；

o Client Key Exchange：客户端向服务端发送自己生成的公钥（此公钥为椭圆曲线的公钥，用于协商出对称加密的密钥）；

o Change Cipher Spec：变更密码规范；告知服务端/客户端，以后的通信都是基于AES加密的；

o Encrypted Handshake Message：基于协商生成的密钥，用AES加密验证信息让服务端/客户端进行认证；如果对方可以解密，则双方认证无误开始通信；

o New Session Ticket：是优化SSL连接的一种方法，此处不做特别说明

<br/>


***
## 四、nginx 	ssl_session_cache明细

![image](https://user-images.githubusercontent.com/29038574/164047109-5e9785a9-51be-46e2-bcc2-e1c398ef26b4.png)

参考：

https://www.csdn.net/tags/Ntjagg2sNDA3ODItYmxvZwO0O0OO0O0O.html （TLS四次握手）

https://blog.csdn.net/u010285974/article/details/85320788 （HTTPS的七次握手（TCP三次+TLS四次））

https://blog.csdn.net/m0_50084718/article/details/113377136  (用wireshark抓包分析TLS协议)

http://nginx.org/en/docs/mail/ngx_mail_ssl_module.html (nginx 详细参数配置)
