## 0、重定向基本流程

![](https://www.icode9.com/i/l/?n=20&i=blog/1158910/202012/1158910-20201204181234377-626967483.png)

## 1、301 、302 重定向区别

### 1.0 初始配置

```
     server {
         listen 90;
         server_name _;
         location /test {
             default_type application/json;
             return http://www.vivo.com;
         }
     }
```

浏览器请求：**http://10.101.192.176:90/test**

会跳转到：**http://www.vivo.com/**

再跳转到：“**https://www.vivo.com/**

![](https://github.com/No-sleeping/self-learn/blob/main/images/other/nginx%20redirect/1.png)

### 1.1 修改return(默认302跳转)

```
     server {
         listen 90;
         server_name _;
         location /test {
             default_type application/json;
             return http://www.iqoo.com;
         }
     }
```

**会跳转到：http://www.iqoo.com**

**再跳转到：https://www.iqoo.com**

**{{这里放一张}}**

<br/>

### 1.2 修改为301 跳转 到iqoo（状态码变化302--301）

```
     server {
         listen 90;
         server_name _;
         location /test {
             default_type application/json;
             return 301 http://www.iqoo.com;
         }
     }
```

效果同 1.1，状态码改为**301**

**{{这里放一张}}**

### 1.3 修改return ，再请求，依旧到iqoo

```
     server {
         listen 90;
         server_name _;
         location /test {
             default_type application/json;
            # return 301 http://www.vivo.com;
             return  http://www.vivo.com;
         }
     }
```

效果通1.2

**{{这里放一张}}**

### 1.4 **清除缓存**之后再请求 ，到vivo（chrome自带功能停用缓存）

会跳转到：**http://www.vivo.com/**

再跳转到：“**https://www.vivo.com/

**{{这里放一张}}**

<br/>

## 2.0  301、308 区别（是否方法保持）

```
    server {
        access_log  logs/test.access.log  main;
        listen 90;
        server_name _;
        location /test {
#            return 301 http://10.101.192.176:100;
            return 308 http://10.101.192.176:100;
        }


    server {
        access_log  logs/100.access.log  main;
        listen 100;
        server_name www.100.com;
        location / {
            default_type application/json;
            return 200 "this is in 100.com";
        }
    }
```

**301：**

```
{{ip}} - - [05/Sep/2022:16:02:27 +0800] "GET / HTTP/1.1" 200 18 "http://{{ip}} :90/testwq" "PostmanRuntime/7.26.8" "-" "-"

{{ip}}  - - [05/Sep/2022:16:02:27 +0800] "POST /testwq HTTP/1.1" 301 169 "-" "PostmanRuntime/7.26.8" "-" "-"
```

**308：**

```
{{ip}}  - - [05/Sep/2022:16:05:19 +0800] "POST / HTTP/1.1" 200 18 "http://{{ip}} :90/testwq" "PostmanRuntime/7.26.8" "-" "-"

{{ip}}  - - [05/Sep/2022:16:05:19 +0800] "POST /testwq HTTP/1.1" 308 171 "-" "PostmanRuntime/7.26.8" "-" "-"
```

<br/>

## 3.0 总结

||永久重定向|临时重定向|
|--|--|--|
|方法保持（POST--POST）|308|307|
|方法不保持（POST--get）|301|302、303|

**HTTP301** 状态码代表的意思是 **永久重定向**，即 HTTP 301 Moved Permanently 响应状态。

**HTTP302** 状态码代表的意思是 **临时重定向**，即 HTTP 302 Found 响应状态。

**HTTP303** 状态码代表的意思是 **当前请求的资源在其它地址**，即 HTTP 303 See Other 响应状态。

**HTTP304 **状态码代表的意思是 **请求资源与本地缓存相同，未修改**，即 HTTP 304 Not Modified 响应状态。

**HTTP307** 状态码代表的意思是 临时**重定向，同302**，即 HTTP 307 Temporary Redirect 响应状态。

**HTTP308**** 状态码代表的意思是 永久**重定向，同301**，即 HTTP 308 Permanent Redirect 响应状态。
