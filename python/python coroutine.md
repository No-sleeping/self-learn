## Python协程的发展时间较长：

- Python2.5 为生成器引用.send()、.throw()、.close()方法
- Python3.3 为引入yield from，可以接收返回值，可以使用yield from定义协程
- Python3.4 加入了asyncio模块
- Python3.5 增加async、await关键字，在语法层面的提供支持
- Python3.7 使用async def + await的方式定义协程
- 此后asyncio模块更加完善和稳定，对底层的API进行的封装和扩展
- Python将于3.10版本中移除以yield from的方式定义协程
  
  <br/>

# **协程**（Coroutine，又称微线程，纤程）

是一种比线程更加轻量级的存在，**协程不是被操作系统内核所管理，而完全是由程序所控制。**
如同一个进程可以有很多线程一样，一个线程可以有很多协程。

但是，协程不是被操作系统所管理的，没有改变CPU最小执行单元是线程，协程是完全由程序所控制的（用户态执行），不会产生上下文切换。

协程具有高并发、高扩展性、低成本的特点，一个CPU支持上万的协程都不是问题。所以，很适合用于高并发处理。通常，协程可以处理IO密集型程序的效率问题，但是处理CPU密集型不是它的长处，如要充分发挥CPU利用率可以结合多进程+协程。

<br/>

## 优点

- 无需线程上下文切换的开销
- 无需原子操作锁定及同步的开销
- 方便切换控制流，简化编程模型

## 缺点

- 无法利用多核资源：协程的本质是个单线程，它不能同时在多个核用上。协程需要和进程配合才能运行在多CPU上。当然我们日常所编写的绝大部分应用都没有这个必要，除非是cpu密集型应用。
- 进行阻塞（Blocking）操作（如IO时）会阻塞掉整个程序。

<br/>

```python
import asyncio
import time
import requests
import aiohttp


async def run(x):
    subStartTime = time.time()

    n = 100000000
    while True:
        n -= 1
        if n < 0:
            break

    subEndTime = time.time()
    print("subFunction during time: %s" %(subEndTime-subStartTime))


if __name__ == '__main__':
    startTime = time.time()

    tasks = [asyncio.ensure_future(run(i)) for i in range(5)]
    loop = asyncio.get_event_loop()
    loop.run_until_complete(asyncio.wait(tasks))

    endTime = time.time()
    print("during time: %s" % (endTime-startTime))
    
----
subFunction during time: 6.079732894897461
subFunction during time: 6.064803600311279
subFunction during time: 6.074823617935181
subFunction during time: 6.077428102493286
subFunction during time: 6.097795724868774
during time: 30.39854383468628

Process finished with exit code 0
```

```python


def run():
    print(1)
    yield
    print(2)
    yield

x = run()
for _ in range(3):
    next(x)

---
Traceback (most recent call last):
  File "*****/iterator.py", line 43, in <module>
    next(x)
StopIteration
1
2

```

<br/>

<br/>

https://juejin.cn/post/7027998293351202853#heading-13 （浅析Python的进程、线程与协程）

https://juejin.cn/post/7027610930431131685 (一分钟明白IO密集型与CPU密集型的区别)

https://docs.python.org/3/library/asyncio-task.html

https://cuiqingcai.com/6160.html
