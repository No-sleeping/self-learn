# ServiceMesh

## 一、概念

> service mesh is a dedicated infrastructure layer for handling service-to-service communication. It’s responsible for the reliable delivery of requests through the complex topology of services that comprise a modern, cloud native application. In practice, the服务网格is typically implemented as an array of lightweight network proxies that are deployed alongside application code, without the application needing to be aware.

服务网格（Service Mesh）是处理服务间通信的基础设施层。它负责构成现代云原生应用程序的复杂服务拓扑来可靠地交付请求。在实践中，Service Mesh 通常以轻量级网络代理阵列的形式实现，这些代理与应用程序代码部署在一起，对应用程序来说无需感知代理的存在。

## 二、演进过程

### 2.1原始通信时代

通信需要底层能够传输字节码和电子信号的物理层来完成，在TCP协议出现之前，服务需要自己处理网络通信所面临的丢包、乱序、重试等一系列流控问题，因此服务实现中，除了业务逻辑外，还夹杂着对网络传输问题的处理逻辑。

![](https://img-blog.csdnimg.cn/20200713112445636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 2.2 TCP时代

为了避免每个服务都需要自己实现一套相似的网络传输处理逻辑，TCP协议出现了，它解决了网络传输中通用的流量控制问题，将技术栈下移，从服务的实现中抽离出来，成为操作系统网络层的一部分。

![](https://img-blog.csdnimg.cn/20200713111145737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 2.3 第一代微服务

在TCP出现之后，机器之间的网络通信不再是一个难题，以GFS/BigTable/MapReduce为代表的分布式系统得以蓬勃发展。

这时，分布式系统特有的通信语义又出现了，如熔断策略、负载均衡、服务发现、认证和授权、quota限制、trace和监控等等，于是服务根据业务需求来实现一部分所需的通信语义。

![](https://img-blog.csdnimg.cn/20200713111239314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 2.4 第二代微服务

为了避免每个服务都需要自己实现一套分布式系统通信的语义功能，随着技术的发展，一些面向微服务架构的开发框架出现了，如Twitter的Finagle、Facebook的Proxygen以及Spring Cloud等等，这些框架实现了分布式系统通信需要的各种通用语义功能：如负载均衡和服务发现等，因此一定程度上屏蔽了这些通信细节，使得开发人员使用较少的框架代码就能开发出健壮的分布式系统。

![](https://img-blog.csdnimg.cn/20200713111325171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

<br/>

### 2.5 第一代service mesh

第二代微服务模式看似完美，但开发人员很快又发现，它也存在一些本质问题：

- 其一，虽然框架本身屏蔽了分布式系统通信的一些通用功能实现细节，但开发者却要花更多精力去掌握和管理复杂的框架本身，在实际应用中，去追踪和解决框架出现的问题也绝非易事；
- 其二，开发框架通常只支持一种或几种特定的语言，回过头来看文章最开始对微服务的定义，一个重要的特性就是语言无关，但那些没有框架支持的语言编写的服务，很难融入面向微服务的架构体系，想因地制宜的用多种语言实现架构体系中的不同模块也很难做到；
- 其三，框架以lib库的形式和服务联编，复杂项目依赖时的库版本兼容问题非常棘手，同时，框架库的升级也无法对服务透明，服务会因为和业务无关的lib库升级而被迫升级。

因此以Linkerd，Envoy，Ngixmesh为代表的代理模式（边车模式）应运而生，这就是第一代Service Mesh，它将分布式服务的通信抽象为单独一层，在这一层中实现负载均衡、服务发现、认证授权、监控追踪、流量控制等分布式系统所需要的功能，作为一个和服务对等的代理服务，和服务部署在一起，接管服务的流量，通过代理之间的通信间接完成服务之间的通信请求，这样上边所说的三个问题也迎刃而解。

![](https://img-blog.csdnimg.cn/20200713112625152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

如果我们从一个全局视角来看，就会得到如下部署图：

![](https://img-blog.csdnimg.cn/20200713112648894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

如果我们暂时略去服务，只看Service Mesh的单机组件组成的网络

![](https://img-blog.csdnimg.cn/20200713112702754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

<br/>

### 2.6 第二代service mesh

第一代Service Mesh由一系列独立运行的单机代理服务构成，为了提供统一的上层运维入口，演化出了集中式的控制面板，所有的单机代理组件通过和控制面板交互进行网络拓扑策略的更新和单机数据的汇报。这就是以Istio为代表的第二代Service Mesh。

在新一代的ServiceMesh架构中(下图上方)，服务的消费方和提供方主机(或者容器)两边都会部署代理SideCar。ServiceMesh比较正式的术语也叫数据面板(DataPlane)，与数据面板对应的还有一个独立部署的控制面板(ControlPlane)，用来集中配置和管理数据面板，也可以对接各种服务发现机制(如K8S服务发现)。术语数据面板和控制面板，估计是偏网络SDN背景的人提出来的。

![](https://img-blog.csdnimg.cn/20200713105609546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

在Service Mesh架构中，给每一个微服务实例部署一个Sidecar Proxy。该Sidecar Proxy负责接管对应服务的入流量和出流量，并将微服务架构中的服务订阅、服务发现、熔断、限流、降级、分布式跟踪等功能从服务中抽离到该Proxy中。

Sidecar以一个独立的进程启动，可以每台宿主机共用同一个Sidecar进程，也可以每个应用独占一个Sidecar进程。所有的服务治理功能，都由Sidecar接管，应用的对外访问仅需要访问Sidecar即可。当该Sidecar在微服务中大量部署时，这些Sidecar节点自然就形成了一个服务网格。

![](https://img-blog.csdnimg.cn/20200713113106416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWNob3VmZWk5MA==,size_16,color_FFFFFF,t_70)

### 2.7 两代service  mesh 介绍

第一代Service Mesh的代表为*Linkerd*和*Envoy*。Linkerd基于Twitter的Fingle，使用Scala编写，是业界第一个开源的Service Mesh方案，在长期的实际生产环境中获得验证。Envoy底层基于**C++**，性能上优于使用Scala的Linkerd。同时，Envoy社区成熟度较高，商用稳定版本面世时间也较长。这两个开源实现都是以Sidecar为核心，绝大部分关注点都是如何做好Proxy，并完成一些通用控制面的功能。但是当你在容器中大量部署Sidecar以后，如何管理和控制这些Sidecar本身就是一个不小的挑战。

第二代Service Mesh主要改进集中在更加强大的控制面功能（与之对应的Sidecar Proxy被称之为数据面），典型代表有*Istio和Conduit*。Istio是Google、IBM和Lyft合作的开源项目，是目前最主流的Service Mesh方案，也是事实上的第二代Service Mesh标准。在Istio中，直接把Envoy作为Sidecar。除了Sidecar，Istio中的控制面组件都是使用**Go**语言编写。

<br/>

<br/>

# Serverless

## 概念

> Serverless computing is a method of providing backend services on an as-used basis. A Serverless provider allows users to write and deploy code without the hassle of worrying about the underlying infrastructure. A company that gets backend services from a serverless vendor is charged based on their computation and do not have to reserve and pay for a fixed amount of bandwidth or number of servers, as the service is auto-scaling. Note that although called serverless, physical servers are still used but developers do not need to be aware of them.

**无服务器计**算****是一种**按需提供后端服务**的方法。无服务器提供程序允许用户编写和部署代码，而不必担心底层基础结构。从无服务器供应商处获得后端服务的公司将根据其计算费用，而不必保留和支付固定数量的带宽或服务器数量，因为该服务是自动扩展的。请注意，尽管称为无服务器，但仍使用物理服务器，但开发人员无需了解它们。

**Serverless = FaaS + BaaS**

### FaaS（Function as a Service，函数即服务）

FaaS意在无须自行管理服务器系统或自己的服务器应用程序，即可直接运行后端代码。其中所指的服务器应用程序，是该技术与容器和PaaS（平台即服务）等其他现代化架构最大的差异。

FaaS可以取代一些服务处理服务器（可能是物理计算机，但绝对需要运行某种应用程序），这样不仅不需要自行供应服务器，也不需要全时运行应用程序。

### BaaS（Backend as a Service，后端即服务）

首先BaaS并非PaaS，它们的区别在于：PaaS需要参与应用的生命周期管理，BaaS则**仅仅提供应用依赖的第三方服务**。典型的PaaS平台需要提供手段让开发者部署和配置应用，例如自动将应用部署到Tomcat容器中，并管理应用的生命周期。BaaS不包含这些内容，BaaS只以API的方式提供应用依赖的后端服务，例如数据库和对象存储。BaaS可以是公共云服务商提供的，也可以是第三方厂商提供的。其次从功能上讲，BaaS可以看作PaaS的一个子集，即提供第三方依赖组件的部分。

BaaS服务还允许我们依赖其他人已经实现的应用逻辑。对于这点，认证就是一个很好的例子。很多应用都要自己编写实现注册、登录、密码管理等逻辑的代码，而对于不同的应用这些代码往往大同小异。完全可以把这些重复性的工作提取出来，再做成外部服务，而这正是Auth0和Amazon Cognito等产品的目标。它们能实现全面的认证和用户管理，开发团队再也不用自己编写或者管理实现这些功能的代码。

<br/>

---

参考：

https://blog.csdn.net/cc18868876837/article/details/90672971（看懂 Serverless，这一篇就够了）

https://blog.csdn.net/baichoufei90/article/details/107293203（ServiceMesh和Serverless）
