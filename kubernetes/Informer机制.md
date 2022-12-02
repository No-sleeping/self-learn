# 0、原理图

![](https://res.cloudinary.com/dqxtn0ick/image/upload/v1555472372/article/code-analysis/informer/client-go.png)

![](https://res.cloudinary.com/dqxtn0ick/image/upload/v1555479782/article/code-analysis/informer/client-go-controller-interaction.jpg)

![](https://raw.githubusercontent.com/NoicFank/picture/main/deltaFIFO/framework.png)

# 1、组件简介

- **SharedIndexInformer**：内部包含controller和Indexer，手握控制器和存储，并实现了sharedIndexInformer共享机制
- **Reflector**：这是远端（APiServer）和本地（DeltaFIFO、Indexer、Listener）之间数据同步逻辑的核心，通过ListAndWatch方法来实现。它提供一个非常重要的**ListAndWatch**方法
- **DeltaFIFO**：Reflector中存储待处理obj(确切说是Delta)的地方，存储本地最新的数据，提供数据Add、Delete、Update方法，以及执行relist的Replace方法
- **Indexer(Local Store)**：Indexer(local store)是Informer机制中本地**最全的数据存储**，其通过DeltaFIFO中最新的Delta不停的更新自身信息，同时需要在本地(DeltaFIFO、Indexer、Listener)之间执行同步，以上两个更新和同步的步骤都由Reflector的ListAndWatch来触发。同时在本地crash，需要进行replace时，也需要查看到Indexer中当前存储的所有key。
- **HandleDeltas**：消费DeltaFIFO中排队的Delta，同时更新给Indexer，并通过distribute方法派发给对应的Listener集合
- **workqueue**：回调函数处理得到的obj-key需要放入其中，待worker来消费，支持延迟、限速、去重、并发、标记、通知、有序。

# 2、数据流向说明

这部分为Informer机制中数据同步的核心思路。需要知道有四类数据存储需要同步：**ApiServer、DeltaFIFO、Listener、Indexer**。对于这四部分，可以简单理解：**Apiserver侧为最权威的数据、DeltaFIFO为本地最新的数据、Indexer为本地最全的数据、Listener为用户侧做逻辑用的数据。**在这其中，存在两条同步通路，一条为远端与本地之间的通路，另一条为本地内部的通路，接下来，让我们对这两条通路进行详细的理解。

## 远端通路：远端(ApiServer) ⇔ 本地(DeltaFIFO、Indexer、Listener)

远端通路可以理解为两类：

第一类为通过List行为产生的同步行为，这类event的DeltaType为Replaced，同时只有在Reflector初始启动时才会产生。

另一类为通过Watch行为产生的同步行为，对于watch到的Added、Modified、Deleted类型的event，对应的DeltaType为Added、Updated、Deleted。

以上步骤为Reflector的ListAndWatch方法将ApiServer测的obj同步到本地DeltaFIFO中。当对应event的Delta放入DeltaFIFO之后，就通过Controller的HandleDeltas 方法，将对应的Delta更新到Indexer和Listener上。具体更新步骤见：HandleDeltas实现逻辑

## 本地通路：本地(DeltaFIFO、Indexer、SyncingListener）之间同步

本地通路是通过Reflector的ListAndWatch方法中运行一个goroutine来执行定期的Resync操做。

首先通过ShouldResync计算出syncingListener,之后其中的store.Resync从Indxer拉一遍所有objs到DeltaFIFO中(list)，其中的Delta为Sync状态。

如果DeltaFIFO的items中存在该obj，就不会添加该obj的sync delta。之后handleDeltas就会同步DeltaFIFO中的Sync Delta给syncingListeners和Indexer。当然这个过程中，别的状态的delta会被通知给所有的listener和Indexer。

站在Indexer的角度，这也是一种更新到最新状态的过程。站在本地的视角，DeltaFIFO、Indexer、Listener都是从DelataFIFO中接收ApiServer发来最新数据。

<br/>

# 3、 思考

## 3.1 什么时候需要Replace？以及DeltaFIFO中Replaced状态的产生方式？

首先需要知道的是Replaced状态的产生，是由于Reflector从ApiServer中list所有的Obj，这些Obj对应的Delta都会被打上Replaced的DeltaType。那本质上来说，只有一种情况需要list，也就是Reflector**刚启动**的时候，它会通过内部的ListAndWatch函数进行一次list，后续就通过watch event来保证ApiServer和本地之间的同步。

但是，我们平时也听过relist，这种操作，也即是当遇到watch event出错(IO错误)的时候，需要重新去向ApiServer请求一次所有的Obj。这类场景的本质其实就是第一种，因为ListAndWatch是运行在BackoffUntil内的，当ListAndWatch因为非stopChan而发生退出时，就会由BackoffUntil在一定时间后拉起，这是就相当于Reflector刚启动。由此就可以清楚Replaced状态的产生，同它字面的意思一致，**就是用ApiServer测的Obj集合替换本地内容。**

<br/>

## 3.2 在整个k8s体系下，是通过哪些手段减少对kube-apiserver的压力？

### informer机制：

-  维护本地store(Indexer)从而使得 R 操作直接访问Inxer即可。也即是通过obj-key在indexer中直接取到obj。
- ListAndWatch机制，减少与ApiServer的交互，只有在起初通过一次List来全量获取，后续通过watch已增量的方式来更新。

### sharedInformer机制：

singleton模式：同一个资源只有一个informer实例，多个listener来绑定informer，从而实现一种资源的改动，通过一个informer实例，通知给若干个listener。避免多个listener都与ApiServer打交道。

<br/>

## 3.3 kube-apiserver又是通过哪些手段减少对etcd的压力？

如果K8s每次想查看资源对象的状态，都要经历一遍List调用，显然对 API Server 也是一个不小的负担，对此，一个容易想到的方法是使用一个cache作保存，需要获取资源状态时直接调cache，当事件来临时除了响应事件外，也对cache进行刷新。

SharedInformer拥有为多个Controller提供一个**共享cache**的能力，从而避免资源缓存的重复、减小空间开销。除此之外，一个SharedInformer对一种资源只建立一个与API Server的Watch监听，且能够将监听得到的事件分发给下游所有感兴趣的Controller，这也显著地减少了API Server的负载压力。实际上，K8s中广泛使用的都是SharedInformer，Informer则出场甚少。

<br/>

## 3.4 为什么需要提供自定义resync的接口？

从listener角度来看，是为了能够按照业务逻辑来定义个性化的同步时间。比如某些业务只需要一天同步一次，某些业务需要1小时同步一次。
从informer的角度来看，同样的，一些自定义的CRD，可能我们不需要那么频繁的同步，或者也可能需要非常频繁的同步。针对不同的资源类型，工厂默认的时间显然不能满足，因此不同的informer可以定义不同的同步时间。

注意的是：连接同一个informer的listener的同步时间，不能小于informer的同步时间。也即是一定是informer同步了之后，listener才能同步。

<br/>

---

参考：

https://www.jianshu.com/p/cc1444867c70  （基础）

https://herbguo.gitbook.io/client-go/informer#1.1-dai-ma-lian-xi

https://github.com/k8s-club/k8s-club/blob/main/articles/Informer%E6%9C%BA%E5%88%B6%20-%20%E6%A6%82%E8%BF%B0.md#%E5%90%84%E7%BB%84%E4%BB%B6%E7%AE%80%E4%BB%8B  （较全）
