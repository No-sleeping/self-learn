# threading

## run()

run() 方法并不启动一个新线程，就是在主线程中调用了一个普通函数而已。

相同的threadID，且直接调用**MainThread**。

```python
import multiprocessing
import threading
import time


def run(x):
    print("in run,parm is %d" % x )
    print(threading.current_thread().name,threading.current_thread().ident)
    time.sleep(2)
    return None


if __name__ == '__main__':
    startTime = time.time()
    for i in range (5):
        t = threading.Thread(target=run,args=(i,))
        t.run()
    endTime = time.time()
    print("during time : %s" %(endTime-startTime))
```

```python
in run,parm is 0
MainThread 17604
in run,parm is 1
MainThread 17604
in run,parm is 2
MainThread 17604
in run,parm is 3
MainThread 17604
in run,parm is 4
MainThread 17604
during time : 10.002792119979858

Process finished with exit code 0

```

## start

start() 方法是启动一个子线程，线程名就是我们定义的name。

start（）方法来启动线程，真正实现了多线程运行。

threadID 不同，且调用的不同的**Thread-X**

```python
t.start()
```

```python
in run,parm is 0
Thread-1 10940
in run,parm is 1
Thread-2 21484
in run,parm is 2
Thread-3 15588
in run,parm is 3
Thread-4 20932
in run,parm is 4during time : 0.0009701251983642578

Thread-5 16412

Process finished with exit code 0
```

<br/>

添加t.join(timeout=None)

join() 方法也可以理解为 提供当前线程阻塞，等待线程结束后，在执行下一个线程，保护线程通畅有序执行，timeout 指等待时间

```python
t.start()
t.join()
```

```python
in run,parm is 0
Thread-1 8356
in run,parm is 1
Thread-2 14968
in run,parm is 2
Thread-3 12544
in run,parm is 3
Thread-4 19712
in run,parm is 4
Thread-5 22088
during time : 10.006307125091553

Process finished with exit code 0
```

<br/>

# multiprocessing

## run()

同thread。

```python
import multiprocessing
import threading
import time


def run(x):
    print("in run,parm is %d" % x )
    print(multiprocessing.current_process().name,multiprocessing.current_process().ident)
    time.sleep(2)
    return None


if __name__ == '__main__':
    startTime = time.time()
    for i in range(5):
        p = multiprocessing.Process(target=run,args=(i,))
        p.run()
        
        # p.start()
        # p.join()
    endTime = time.time()
    print("during time : %s" %(endTime-startTime))
```

```python
in run,parm is 0
MainProcess 21424
in run,parm is 1
MainProcess 21424
in run,parm is 2
MainProcess 21424
in run,parm is 3
MainProcess 21424
in run,parm is 4
MainProcess 21424
during time : 10.00246524810791

Process finished with exit code 0
```

<br/>

## start()

```python
p.start()
```

```python
during time : 0.054853200912475586
in run,parm is 0
Process-1 22108
in run,parm is 1
Process-2 22180
in run,parm is 2
Process-3 13920
in run,parm is 3
Process-4 4212
in run,parm is 4
Process-5 14192

Process finished with exit code 0
```

```python
p.start()
p.join()
```

```python
in run,parm is 0
Process-1 18428
in run,parm is 1
Process-2 4484
in run,parm is 2
Process-3 22260
in run,parm is 3
Process-4 1272
in run,parm is 4
Process-5 21592
during time : 10.811495304107666

Process finished with exit code 0
```

<br/>

<br/>

---

<br/>

单函数执行：5s

```python
if __name__ == '__main__':
    startTime = time.perf_counter()
    run(1)
    endTime = time.perf_counter()
    print("during time : %s" %(endTime-startTime))


# time python runpy.py 
MainProcessstart at 1677138713.65
MainProcess is done
during time : 5.04020094872

real	0m5.060s
user	0m5.053s
sys	0m0.007s
```

<br/>

10进程执行：5s

p.start()

```python
import multiprocessing
import threading
import time
import datetime


def run(x):
    startTime = time.time()
    print(multiprocessing.current_process().name+'start at %s' % startTime )
    n = 100000000
    while True:
        n -= 1
        if n < 0:break
    print("%s is done" % multiprocessing.current_process().name)

    return None


if __name__ == '__main__':
    startTime = time.perf_counter()
    for i in range(10):
        p = multiprocessing.Process(target=run,args=(i,))
        p.start()
    endTime = time.perf_counter()
    print("during time : %s" %(endTime-startTime))
```

```sh
# time python runpy.py 
Process-1start at 1677141996.12
Process-2start at 1677141996.12
Process-3start at 1677141996.12
Process-4start at 1677141996.12
Process-5start at 1677141996.12
Process-6start at 1677141996.12
Process-7start at 1677141996.12
Process-8start at 1677141996.12
during time : 0.00515794754028
Process-9start at 1677141996.12
Process-10start at 1677141996.12
Process-10 is done
Process-2 is done
Process-6 is done
Process-7 is done
Process-3 is done
Process-5 is done
Process-8 is done
Process-1 is done
Process-4 is done
Process-9 is done

real	0m5.527s
user	0m52.433s
sys	0m0.042s

```

<br/>

10进程执行：50s

p.run()

```python
if __name__ == '__main__':
    startTime = time.perf_counter()
    for i in range(10):
        p = multiprocessing.Process(target=run,args=(i,))
        p.run()
    endTime = time.perf_counter()
    print("during time : %s" %(endTime-startTime))
```

<br/>

```sh
# time python runpy.py 
MainProcessstart at 1677142184.82
MainProcess is done
MainProcessstart at 1677142189.87
MainProcess is done
MainProcessstart at 1677142194.86
MainProcess is done
MainProcessstart at 1677142199.85
MainProcess is done
MainProcessstart at 1677142204.87
MainProcess is done
MainProcessstart at 1677142209.9
MainProcess is done
MainProcessstart at 1677142214.89
MainProcess is done
MainProcessstart at 1677142219.88
MainProcess is done
MainProcessstart at 1677142224.92
MainProcess is done
MainProcessstart at 1677142229.96
MainProcess is done
during time : 50.1973679066

real	0m50.218s
user	0m50.203s
sys	0m0.014s
```
