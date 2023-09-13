1. defer代码的延迟顺序与最终的执行顺序是反向的。
2. defer 会在当前函数返回之前执行 defer 注册的函数
3. panic 场景依然会被调用
4. 配套的两个行为代码可以放在最近的位置：创建&释放、加锁&放锁、前置&后置，使得代码更易读，编程体验优秀。

<br/>

<br/>

demo1

```golang
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}

---
result = 0
result ++ // 这里defer函数内没有额外参数,++的就是要返回的result 
return result  // result = 1
```

demo2

```golang
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}

---
r = t //r= 5
t = t+5 // t= 10
return r  //return r=5
```

demo3

```golang
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}

---
r = 1
     /**
     * 这里修改的r是函数形参的值，是外部传进来的
     * func(r int){}里边r的作用域只该func内，修改该值不会改变func外的r值
     */
r = r +5  // 这里的r和f要返回的r不是一个地址
return r // r = 1，要返回的r，一直没变。
```

<br/>

<br/>

<br/>

https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html

https://zhuanlan.zhihu.com/p/351176808

https://www.cnblogs.com/roadwide/p/17208357.html
