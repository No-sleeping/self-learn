# 0、网络IO发展模型

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1641f3a4b87474fbca3abbbe65846ea~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 0.1 阻塞IO

### 0.1.1 单线程阻塞

**只有一个线程在处理。**

> eg：
client1、client2** 同时**对 server1 发起链接请求
server1 同时只能接受一个client1（client2 此时处于等待状态）
当client1断开连接后，client2才能被server1 接受

通过以上流程，我们很容易发现这个过程的缺陷，服务器每次只能处理一个连接请求，CPU没有得到充分利用，性能比较低。如何充分利用CPU的多核特性呢？自然而然的想到了——多线程逻辑。

### 0.1.2 多线程阻塞

> eg：
client1、client2** 同时**对 server1 发起链接请求
server1 能同时接受一个client1、client2

我们用**多线程**解决了，服务器同时只能处理一个请求的问题，但同时又带来了一个问题，如果客户端连接比较多时，服务端会创建大量的线程来处理请求，但线程本身是比较耗资源的，创建、上下文切换都比较**耗资源**，又如何去解决呢？

## 0.2 非阻塞IO

如果我们把所有的Socket（文件句柄，后续用Socket来代替fd的概念，尽量减少概念，减轻阅读负担）都放到队列里，只用一个线程来轮训所有的Socket的状态，如果准备好了就把它拿出来，是不是就减少了服务端的线程数呢？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4397a43fb7e7493bb4e6d5697a503cbd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

<br/>

服务端专门有一个线程来负责轮询所有的Socket，来确认操作系统是否完成了相关事件，如果有则返回处理，如果无继续轮询，大家一起来思考下？此时又带来了什么问题呢。

CPU的空转、系统调用（每次轮询到涉及到一次系统调用，通过内核命令来确认数据是否准备好），造成资源的浪费，那有没有一种机制，来解决这个问题呢？

## 0.3 多路复用IO

server端将轮询的方式转变为“由事件触发”

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52718fe22db342c98f0c9c8406588d6b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

# 1、NIO（非阻塞）

## 1.1 单Reactor单线程模型

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b91d11bd0b424cbc9f43f54938c3bea0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

<br/>

## 1.2 单Reactor多线程模型

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fb5609a611d4050b7e783f8d31398a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

<br/>

## 1.3 多Reactor多线程模型

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9413d22eb9fc4e47ac25573f622fdd94~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

||处理流程|优点|缺点|
|:--|--|--|--|
|单Reactor单线程模型|Reactor监听连接事件、Socket事件，当有连接事件过来时交给Acceptor处理，当有Socket事件过来时交个对应的Handler处理。|所有的处理过程都在一个线程里；各模块解耦|无法发挥cpu多核优势；大流量时可能出现瓶颈|
|单Reactor多线程模型|Reactor、Acceptor作用不变，Handler完成读事件后，包装成一个任务对象，交个线程池处理。|利用cpu多核优势；|当大流量时，处理读写任务的Reactor可能出现性能瓶颈|
|多Reactor多线程模型|这种模型相对单Reactor多线程模型，只是将Scoket的读写处理从mainReactor中拎出来，交给subReactor线程来处理。    |让主线程专注于连接事件的处理，子线程专注于读写事件吹，从设计上进一步解耦；|实现上会比较复杂，在极度追求单机性能的场景中可以考虑使用。|
|-|mainReactor主线程负责连接事件的监听和处理，当Acceptor处理完连接过程后，主线程将连接分配给subReactor；|利用CPU多核的优势。||
|-|subReactor负责mainReactor分配过来的Socket的监听和处理，当有Socket事件过来时交个对应的Handler处理；|||

<br/>

# 2、Redis线程模型

## 2.1 模型介绍

IO多路复用负责各事件的监听（连接、读、写等），当有事件发生时，将对应事件放入队列中，由事件分发器根据事件类型来进行分发；

如果是连接事件，则分发至连接应答处理器；GET、SET等redis命令分发至命令请求处理器。

命令处理完后产生命令回复事件，再由事件队列，到事件分发器，到命令回复处理器，回复客户端响应。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7f9e3dc62d345f39c1b72a21922549d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 2.2 客户端和服务端交互流程

### 2.2.1 连接流程

- Redis服务端主线程监听固定端口，并将连接事件绑定连接应答处理器。
- 客户端发起连接后，连接事件被触发，IO多路复用程序将连接事件包装好后丢人事件队列，然后由事件分发处理器分发给连接应答处理器。
- 连接应答处理器创建client对象以及Socket对象，我们这里关注Socket对象，并产生ae_readable事件，和命令处理器关联，标识后续该Socket对可读事件感兴趣，也就是开始接收客户端的命令操作。
- 当前过程都是由一个主线程负责处理。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0527d41b5e614c0caecd56fe653ecfef~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

<br/>

### 2.2.2 命令处理流程

- 客户端发起SET命令，IO多路复用程序监听到该事件后（读事件），将数据包装成事件丢到事件队列中（事件在上个流程中绑定了命令请求处理器）；
- 事件分发处理器根据事件类型，将事件分发给对应的命令请求处理器；
- 命令请求处理器，读取Socket中的数据，执行命令，然后产生ae_writable事件，并绑定命令回复处理器；
- IO多路复用程序监听到写事件后，将数据包装成事件丢到事件队列中，事件分发处理器根据事件类型分发至命令回复处理器；
- 命令回复处理器，将数据写入Socket中返回给客户端。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6e9f1ea3ad4479a865e195cb2ad3232~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 2.3 redis使用模型

**Redis采用的是单线程Reactor模型**

那 单线程Reactor 的优缺点，前面已经介绍过了。只要是使用该模型，就已经遇到前述的比如不能利用多核cpu的问题。

于是在redis6.0 的时候 redis 引入了多线程（Background IO）的概念。

主要执行流程：

- 客户端发送请求命令，触发读就绪事件，服务端主线程将Socket（为了简化理解成本，统一用Socket来代表连接）放入一个队列，主线程不负责读；
- IO 线程通过Socket读取客户端的请求命令，主线程忙轮询，等待所有 I/O 线程完成读取任务，IO线程只负责读不负责执行命令；
- 主线程一次性执行所有命令，执行过程和单线程一样，然后需要返回的连接放入另外一个队列中，有IO线程来负责写出（主线程也会写）；
- 主线程忙轮询，等待所有 I/O 线程完成写出任务。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdccd51a4b8b456baec41659c268e7b5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

<br/>

# 3、redis6.0 BIO

https://juejin.cn/post/6949929673615179789?searchId=20231206151621446D0A75540FA7DA438B#heading-19

<br/>

# 4、redis 单线程？

https://strikefreedom.top/archives/multiple-threaded-network-model-in-redis#toc-head-7

---

https://juejin.cn/post/7036178412825755679#heading-6 （Redis线程模型的前世今生）

https://juejin.cn/post/6949929673615179789?searchId=20231206151621446D0A75540FA7DA438B#heading-0（Redis多线程架构的演进）

https://xie.infoq.cn/article/d01329aceeed184a8e71575ff （Redis 究竟是单线程还是多线程？）
