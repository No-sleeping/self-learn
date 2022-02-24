前言：

```
洛：大爷，楼上322住的是马冬梅家吧？
 
大爷：马都什么？
 
夏洛：马冬梅。
 
大爷：什么都没啊？
 
夏洛：马冬梅啊。
 
大爷：马什么没？
 
夏洛：行，大爷你先凉快着吧.
```

在了解这三个概念之前我们先要了解**HTTP是无状态**的Web服务器，

# cookie

生成过程：

- 浏览器第一次访问服务端时，服务器此时肯定不知道他的身份，所以创建一个独特的身份标识数据，格式为key=value，放入到Set-Cookie字段里，随着响应报文发给浏览器。
- 浏览器看到有Set-Cookie字段以后就知道这是服务器给的身份标识，于是就保存起来，下次请求时会自动将此key=value值放入到Cookie字段中发给服务端。
- 服务端收到请求报文后，发现Cookie字段中有值，就能根据此值识别用户的身份然后提供个性化的服务。

![](https://imgconvert.csdnimg.cn/aHR0cDovL3AxLnBzdGF0cC5jb20vbGFyZ2UvcGdjLWltYWdlL2NkZjhlZDhjOGU4ZjRlN2NhNzEzYjM1NzU4YjE1OTA0?x-oss-process=image/format,png)

<br/>

# session

- 当程序需要为某个客户端的请求创建一个session时，服务器首先检查这个客户端的请求里是否已包含了一个session标识------------称为session id，如果已包含则说明以前已经为此客户端创建过session，服务器就按照session id把这个session检索出来使用（检索不到，会新建一个）。

- 如果客户端请求不包含session id，则为此客户端**创建一个session并且生成一个与此session相关联的session id**，这个session id将被在本次响应中返回给客户端保存。

**保存这个session id的方式可以采用cookie**，这样在交互过程中浏览器可以自动的按照规则把这个标识发挥给服务器。一般这个cookie的名字都是类似于SEEESIONID。

![](https://img-blog.csdnimg.cn/20181126151900484.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_86,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMwMzQyMjY=,size_20,color_FFFFFF,t_50)

```
Jsessionid?
Jsessionid只是tomcat的对sessionid的叫法，其实就是sessionid；在其它的容器也许就不叫jsessionid了。
```

## session 弊端

- 服务器压力大
  
  通常session是存储在内存中的，每个用户通过认证之后都会将session数据保存在服务器的内存中，而当用户量增大时，服务器的压力增大。
- CSRF
  
  session是基于cookie进行用户识别的, cookie如果被截获，用户就会很容易受到跨站请求伪造的攻击。
- 扩展性弱
  
  如果将来搭建了多个服务器，虽然每个服务器都执行的是同样的业务逻辑，但是session数据是保存在内存中的（不是共享的），用户第一次访问的是服务器1，当用户再次请求时可能访问的是另外一台服务器2，服务器2获取不到session信息，就判定用户没有登陆过。

## Cookie和Session的区别：

1、cookie数据存放在客户的浏览器上，session数据放在服务器上。
2、cookie不是很安全，别人可以分析存放在本地的cookie并进行cookie欺骗,考虑到安全应当使用session。
3、session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能,考虑到减轻服务器性能方面，应当使用cookie。
4、单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。

# token

token是用户身份的验证方式，我们通常叫它：令牌。

最简单的token组成:`uid(用户唯一的身份标识)、time(当前时间的时间戳)、sign(签名，由token的前几位+盐`以哈希算法压缩成一定长的十六进制字符串，可以防止恶意第三方拼接token请求服务器)。还可以把不变的参数也放进token，避免多次查库。

生成步骤：

- A：当用户首次登录成功（注册也是一种可以适用的场景）之后, 服务器端就会生成一个 token 值，这个值，会在服务器保存token值(保存在数据库中)，再将这个token值返回给客户端.
- B：客户端拿到 token 值之后,进行本地保存。（SP存储是大家能够比较支持和易于理解操作的存储）
- C：当客户端再次发送网络请求(一般不是登录请求)的时候,就会将这个 token 值附带到参数中发送给服务器.
- D：服务器接收到客户端的请求之后,会取出token值与保存在本地(数据库)中的token值做对比
  
  对比一：如果两个 token 值相同， 说明用户登录成功过!当前用户处于登录状态!
  
  对比二：如果没有这个 token 值, 则说明没有登录成功.
  
  对比三：如果 token 值不同: 说明原来的登录信息已经失效,让用户重新登录.

![](https://img-blog.csdnimg.cn/20181126161842605.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_90,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMwMzQyMjY=,size_20,color_FFFFFF,t_70)

<br/>

## Token 和 Session 的区别：

- 认证成功后，会对当前用户数据进行加密，生成一个加密字符串token，返还给客户端（服务器端并不进行保存）
- 浏览器会将接收到的token值存储在Local Storage中
- 再次访问时服务器端对token值的处理：服务器对浏览器传来的token值进行解密，解密完成后进行用户数据的查询，如果查询成功，则通过认证。（解决了扩展性弱的问题）

<br/>

参考：

https://www.jianshu.com/p/bd1be47a16c1  （Cookie、Session、Token那点事儿（原创））

https://blog.csdn.net/sinat_34191046/article/details/88740880  (为什么使用token？session与token的区别)
