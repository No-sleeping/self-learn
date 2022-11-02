# 一、概念

## 1.1 单项认证流程

![](https://lark-assets-prod-aliyun.oss-accelerate.aliyuncs.com/lark/0/2020/png/18611/1585034778516-6e938349-9008-4940-b24f-3ceb74f57fd6.png?OSSAccessKeyId=LTAI4GGhPJmQ4HWCmhDAn4F5&Expires=1667353937&Signature=8UurL9foEs0ONQKRIu9Z4kWcQzc%3D&response-content-disposition=inline#alt=undefined)

1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务器端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），取出了服务端公钥；
4. 客户端生成一个随机数（密钥R），用刚才得到的服务器公钥去加密这个随机数形成密文，发送给服务端；
5. 服务端用自己的私钥（server.key）去解密这个密文，得到了密钥R
6. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。

## 1.2 双向认证流程

![](https://lark-assets-prod-aliyun.oss-accelerate.aliyuncs.com/lark/0/2020/png/18611/1585034830354-cf4e77f6-e87c-4bfd-9fb5-e72746f2dcd1.png?OSSAccessKeyId=LTAI4GGhPJmQ4HWCmhDAn4F5&Expires=1667353979&Signature=hcD4NYSRv4A7g3pOtbP1bAB%2FLwI%3D&response-content-disposition=inline#alt=undefined)

1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），取出了服务端公钥；
4. **客户端将客户端公钥证书（client.crt）发送给服务器端；**
5. **服务器端使用根证书（root.crt）解密客户端公钥证书，拿到客户端公钥；**
6. 客户端发送自己支持的加密方案给服务器端；
7. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密8. 后发送给客户端；
8. 客户端使用自己的私钥解密加密方案，生成一个随机数R，使用服务器公钥加密后传给服务器端；
9. 服务端用自己的私钥去解密这个密文，得到了密钥R
10. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。

## 

# 二、用法

##  2.1 准备证书

![](https://upload-images.jianshu.io/upload_images/2614681-e42ccfdd74cea3a9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### 2.1.1 root证书

```sh
（1）创建根证书私钥：
openssl genrsa -out root.key 1024

（2）创建根证书请求文件：
openssl req -new -out root.csr -key root.key
后续参数请自行填写，下面是一个例子：
Country Name (2 letter code) [XX]:cn
State or Province Name (full name) []:bj
Locality Name (eg, city) [Default City]:bj
Organization Name (eg, company) [Default Company Ltd]:alibaba
Organizational Unit Name (eg, section) []:test
Common Name (eg, your name or your servers hostname) []:root
Email Address []:a.alibaba.com
A challenge password []:
An optional company name []:

（3）创建根证书：
openssl x509 -req -in root.csr -out root.crt -signkey root.key -CAcreateserial -days 3650
```

**注意：**

1. 根证书的Common Name填写root就可以，**所有客户端和服务器端的证书这个字段需要填写域名**，一定要注意的是，根证书的这个字段和客户端证书、服务器端证书不能一样
2. 其他所有字段的填写，根证书、服务器端证书、客户端证书需保持一致
3. 最后的密码可以直接回车跳过。

### 2.1.2 server证书

```sh
（1）生成服务器端证书私钥：
openssl genrsa -out server.key 1024

（2） 生成服务器证书请求文件，过程和注意事项参考根证书，本节不详述：
openssl req -new -out server.csr -key server.key

（3） 生成服务器端公钥证书
openssl x509 -req -in server.csr -out server.crt -signkey server.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
```

### 2.1.3 client证书

```sh
（1）生成客户端证书秘钥：
openssl genrsa -out client.key 1024
openssl genrsa -out client2.key 1024

（2） 生成客户端证书请求文件，过程和注意事项参考根证书，本节不详述：
openssl req -new -out client.csr -key client.key
openssl req -new -out client2.csr -key client2.key

（3） 生客户端证书
openssl x509 -req -in client.csr -out client.crt -signkey client.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
openssl x509 -req -in client2.csr -out client2.crt -signkey client2.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650


（4） 生客户端p12格式证书，需要输入一个密码，选一个好记的，比如123456
openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
openssl pkcs12 -export -clcerts -in client2.crt -inkey client2.key -out client2.p12
```

<br/>

## 2.2 验证

### 2.2.1  nginx 配置

```sh
server {
        listen       443 ssl;
        server_name  www.yourdomain.com;
        ssl                  on;  
        ssl_certificate      /data/sslKey/server.crt;  #server公钥证书
        ssl_certificate_key  /data/sslKey/server.key;  #server私钥
        ssl_client_certificate /data/sslKey/root.crt;  #根证书，可以验证所有它颁发的客户端证书
        ssl_verify_client on;  #开启客户端证书验证  

        location / {
            root   html;
            index  index.html index.htm;
        }
    }

```

注意：

有一点需要注意的就是，**如果客户端证书不是由根证书直接颁发的，配置中还需要加一个配置：ssl_verify_depth 1;**

###  2.2.2 case1 带证书的成功调用

```sh
#--cert指定客户端公钥证书的路径
#--key指定客户端私钥文件的路径
#-k不校验证书的合法性，因为我们用的是自签名证书，所以需要加这个参数
#可以使用-v来观察具体的SSL握手过程

curl --cert ./client.crt --key ./client.key https://integration-fred2.fredhuang.com -k -v
* Rebuilt URL to: https://47.93.245.203/
*   Trying 47.93.245.203...
* TCP_NODELAY set
* Connected to 47.93.245.203 (47.93.245.203) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS handshake, CERT verify (15):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: C=CN; ST=BJ; L=BJ; O=Alibaba; OU=Test; CN=integration-fred2.fredhuang.com; emailAddress=a@alibaba.com
*  start date: Nov  2 01:01:34 2019 GMT
*  expire date: Oct 30 01:01:34 2029 GMT
*  issuer: C=CN; ST=BJ; L=BJ; O=Alibaba; OU=Test; CN=root; emailAddress=a@alibaba.com
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
> GET / HTTP/1.1
> host:integration-fred2.fredhuang.com
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.17.5
< Date: Sat, 02 Nov 2019 02:39:43 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Wed, 30 Oct 2019 11:29:45 GMT
< Connection: keep-alive
< ETag: "5db97429-264"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 47.93.245.203 left intact

```

###  2.2.3 case2 使用client2.crt/client2.key这一套客户端证书来调用服务器端

```sh
curl --cert ./client2.crt --key ./client2.key https://integration-fred2.fredhuang.com -k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

###  2.2.4 case3  不带证书的调用

```sh
curl https://integration-fred2.fredhuang.com -k
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.17.5</center>
</body>
</html>
```

<br/>

# 三、拓展概念


## 3.1 证书链

![](https://pic3.zhimg.com/80/v2-5fd2d3ef627d62a385ca8dfef201fff2_720w.webp)

证书链(certificate chain)可以有任意环节的长度：CA证书包括根CA证书、二级CA证书（中间证书）、三级证书.....（证书链越长，加载速度越慢，信任度越差），以上是对证书链的图解。

## 3.2 证书备用名-SAN

通常的SSL证书是不支持多域名的，一个SSL证书绑定一个域名。entrust针对广大用户对支持多域名的需求，全球独家推出了多域型SSL证书(Multi-DomainSSLCertificate)，也就是说：一个多域型SSL证书可以同时用于同一台物理服务器上的所有网站域名的SSL安全加密。也称为UCC证书或者SANsssl证书。


## 3.3 证书格式

- CSR：证书请求文件，这个并不是证书，而是向证书颁发机构获得签名证书的申请文件
- CER：存放证书文件可以是二进制编码或者BASE64编码
- CRT：证书可以是DER编码，也可以是PEM编码，在linux系统中比较常见
- pem：该编码格式在RFC1421中定义，但他也同样广泛运用于密钥管理，实质上是 Base64 编码的二进制内容
- DER：用于二进制DER编码的证书。这些证书也可以用CER或者CRT作为扩展名
- JKS：java的密钥存储文件,二进制格式,是一种 Java 特定的密钥文件格式， JKS的密钥库和私钥可以用不同的密码进行保护
- p12/PFX：**包含所有私钥、公钥和证书**。其以二进制格式存储，也称为 PFX 文件，在windows中可以直接导入到密钥区，密钥库和私钥用相同密码进行保护

<br/>

# 四、其他

## 4.1 参考

https://www.jianshu.com/p/2b2d1f511959

https://help.aliyun.com/document_detail/160093.html

http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_verify_client

https://zhuanlan.zhihu.com/p/449630806  （OpenSSL 自签证书详解）

https://zhuanlan.zhihu.com/p/414949073 （什么是SSL证书链？）

https://www.anxinssl.com/9801.html （证书链）

## 4.2 工具

证书校验：

https://www.ssleye.com/ssltool/cer_check.html

```sh
openssl x509 -in test.crt -noout -text
```

<br/>

自签名证书生成：

https://www.ssleye.com/ssltool/certs_down.html
