# 0、先说结论

1. 两者都属于k8s控制面组件，其作用都是为了监控资源，使对应资源达到期望状态。
2. Operators 就是 管理CRD 资源的一种 Controller
3. 两者都通过 informer 感知 K8s 资源的变化。

# 1、Controller

Controller 里面最常用的两个组件就是 ` shared (thread-safe) cache` and ` informer`。

Informer 在控制器初始化时被创建，运行在控制器本地，有两大主要作用：

   1、同步特定 k8s 资源或特定CR的本地缓存（属于增量同步，因此不会对 k8s 的 apiserver 造成冲击）
   
   2、watch 资源对象的变化类型事件触发事先注册的控制器钩子函数（最重要的作用）

![](https://octetz.s3.us-east-2.amazonaws.com/k8s-controllers-vs-operators/client-go-flow.png)

## 1.1 informer

# 2、Operators

> An operator is a specialized form of controller.

<br/>

具有以下特质的controller 一般被认为是operator：

- Contains workload-specific knowledge
- Manages workload lifecycle
- Offers a CRD
  
  <br/>

能做什么：

- **Automatic monitoring and alerting **---Operators usually know when their applications aren’t working properly and generate appropriate alerts.
- **Automatic version upgrades over time **---Operators are often capable of automatically applying cluster changes to install new app updates as they become available. This significantly reduces the maintenance burden for ops teams.
- **Installing custom resources**---Operators will add the app’s custom resources to the Kubernetes API server, preparing the cluster to host the workload.
- **Provide auto-scaling **--- An operator with domain-specific knowledge can recognize when the configured replica count is too low to comfortably serve current traffic and spin up new instances to maintain performance.
- **Lifecycle management**---Operators ensure new application instances launch into an environment where all prerequisites are already met. They’ll also perform any needed clean-up after a replica stops.
- **Storage management and backups **---Some operators assist the set up of persistent storage. As they understand their application, they may also make backups before applying a potentially destructive action.

operator 仓库：https://operatorhub.io/

<br/>

<br/>

参考：

https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/

https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/operator/

https://www.howtogeek.com/devops/what-are-kubernetes-controllers-and-operators/

https://joshrosso.com/docs/2019/2019-10-13-controllers-and-operators/
