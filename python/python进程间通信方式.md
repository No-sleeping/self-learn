# 消息队列

消息队列常用于单向交互，消息队列操作简单，用于单向交互最方便。

```python
from multiprocessing import Process, Queue

def f(q):
    q.put([42, None, 'hello'])

if __name__ == '__main__':
    q = Queue()
    p = Process(target=f, args=(q,))
    p.start()
    print(q.get())    # prints "[42, None, 'hello']"
    p.join()
```

# 管道

```python
from multiprocessing import Process, Pipe

def f(conn):
    conn.send([42, None, 'hello'])
    conn.close()

if __name__ == '__main__':
    parent_conn, child_conn = Pipe()
    p = Process(target=f, args=(child_conn,))
    p.start()
    print(parent_conn.recv())   # prints "[42, None, 'hello']"
    p.join()
```

# 共享内存

如上所述，在进行并发编程时，通常最好尽量避免使用共享状态。使用多个进程时尤其如此。

但是，如果你真的需要使用一些共享数据，那么 multiprocessing 提供了两种方法。

```python
from multiprocessing import Process, Value, Array

def f(n, a):
    n.value = 3.1415927
    for i in range(len(a)):
        a[i] = -a[i]

if __name__ == '__main__':
    num = Value('d', 0.0)
    arr = Array('i', range(10))

    p = Process(target=f, args=(num, arr))
    p.start()
    p.join()

    print(num.value)
    print(arr[:])
```

# Manager()

由 **Manager() **返回的管理器对象控制一个服务器进程，该进程保存Python对象并允许其他进程使用代理操作它们。

Manager() 返回的管理器支持类型： list 、 dict 、 Namespace 、 Lock 、 RLock 、 Semaphore 、 BoundedSemaphore 、 Condition 、 Event 、 Barrier 、 Queue 、 Value 和 Array 。

```python
from multiprocessing import Process, Manager

def f1(d, l):
    d[1] = '1'
    d['2'] = 2
    d[0.25] = None
    l.reverse()
    print("进程内参数：",d,l)

if __name__ == "__main__":
    with Manager() as manager:
        d = manager.dict()
        l = manager.list(range(10))
        p = Process(target=f1, args=(d, l))
        print("进程外参数：",d, l)
        p.start()
        p.join()
```

<br/>

**需要注意一个陷阱，即Manager对象无法监测到它引用的可变对象值的修改，需要通过触发__setitem__方法来让它获得通知。而触发__setitem__方法比较直接的办法就是增加一个中间变量，**

```python
def test_manager(m):
    tmp = m[0]
    tmp{"id"} = 2
    m[0] = tmp
    
if __name__ == "__main__":
    m = Manager().list()
    m.append({"id":1})
    p = Process(target=test_manager,args=(m,))
    p.start()
    p.join()
    print m[0]
```
