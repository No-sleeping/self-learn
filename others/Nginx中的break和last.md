```

示例5（增加break）：
server{
    listen 80; 
    server_name test.com;
    root /tmp/123.com;
    
    location / {
        rewrite /1.html /2.html break;
        rewrite /2.html /3.html;
    }
    location /2.html
    {
        rewrite /2.html /a.html;
    }
    location /3.html
    {
        rewrite /3.html /b.html;
    }
}
当请求/1.html，最终会访问/2.html
在location{}内部，遇到break，本location{}内以及后面的所有location{}内的所有指令都不再执行。
```

```

示例6（增加last）:
server{
    listen 80; 
    server_name test.com;
    root /tmp/123.com;
    
    location / {
        rewrite /1.html /2.html last;
        rewrite /2.html /3.html;
    }
    location /2.html
    {
        rewrite /2.html /a.html;
    }
    location /3.html
    {
        rewrite /3.html /b.html;
    }
}
当请求/1.html，最终会访问/a.html
在location{}内部，遇到last，本location{}内后续指令不再执行，而重写后的url再次从头开始，从头到尾匹配一遍规则。
```

<br/>

## 总结：

当rewrite规则在location{}外，break和last作用一样，遇到break或last后，其后续的rewrite/return语句不再执行。但后续有location{}的话，还会近一步执行location{}里面的语句,当然前提是请求必须要匹配该location。

当rewrite规则在location{}里，遇到break后，本location{}与其他location{}的所有rewrite/return规则都不再执行。

当rewrite规则在location{}里，遇到last后，本location{}里后续rewrite/return规则不执行，但重写后的url再次从头开始执行所有规则，哪个匹配执行哪个。

---

参考：https://www.cnblogs.com/oldxu/p/11568223.html
