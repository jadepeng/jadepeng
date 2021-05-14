---
title: APM 原理与框架选型
tags: ["apm","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-08-27 15:59
---
文章作者:jqpeng
原文链接: [APM 原理与框架选型](https://www.cnblogs.com/xiaoqi/p/apm.html)

发些存稿:)

## 0. APM简介

随着微服务架构的流行，一次请求往往需要涉及到多个服务，因此服务性能监控和排查就变得更复杂：

- 不同的服务可能由不同的团队开发、甚至可能使用不同的编程语言来实现
- 服务有可能布在了几千台服务器，横跨多个不同的数据中心


因此，就需要一些可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题，这就是APM系统，全称是（**A**pplication **P**erformance **M**onitor，当然也有叫 **A**pplication **P**erformance **M**anagement tools）

AMP最早是谷歌公开的论文提到的 [Google Dapper](http://bigbully.github.io/Dapper-translation)。Dapper是Google生产环境下的分布式跟踪系统，自从Dapper发展成为一流的监控系统之后，给google的开发者和运维团队帮了大忙，所以谷歌公开论文分享了Dapper。

## 1. 谷歌Dapper介绍

### 1.1 **Dapper的挑战**

在google的首页页面，提交一个查询请求后，会经历什么：

- 可能对上百台查询服务器发起了一个Web查询，每一个查询都有自己的Index
- 这个查询可能会被发送到多个的子系统，这些子系统分别用来处理广告、进行拼写检查或是查找一些像图片、视频或新闻这样的特殊结果
- 根据每个子系统的查询结果进行筛选，得到最终结果，最后汇总到页面上


总结一下：

- 一次全局搜索有可能调用上千台服务器，涉及各种服务。
- 用户对搜索的耗时是很敏感的，而任何一个子系统的低效都导致导致最终的搜索耗时


如果一次查询耗时不正常，工程师怎么来排查到底是由哪个服务调用造成的？

- 首先，这个工程师可能无法准确的定位到这次全局搜索是调用了哪些服务，因为新的服务、乃至服务上的某个片段，都有可能在任何时间上过线或修改过，有可能是面向用户功能，也有可能是一些例如针对性能或安全认证方面的功能改进
- 其次，你不能苛求这个工程师对所有参与这次全局搜索的服务都了如指掌，每一个服务都有可能是由不同的团队开发或维护的
- 再次，这些暴露出来的服务或服务器有可能同时还被其他客户端使用着，所以这次全局搜索的性能问题甚至有可能是由其他应用造成的


从上面可以看出Dapper需要：

- 无所不在的部署，无所不在的重要性不言而喻，因为在使用跟踪系统的进行监控时，即便只有一小部分没被监控到，那么人们对这个系统是不是值得信任都会产生巨大的质疑
- 持续的监控


### 1.2 Dapper的三个具体设计目标

1. **性能消耗低**
    APM组件服务的影响应该做到足够小。**服务调用埋点本身会带来性能损耗，这就需要调用跟踪的低损耗，实际中还会通过配置采样率的方式，选择一部分请求去分析请求路径**。在一些高度优化过的服务，即使一点点损耗也会很容易察觉到，而且有可能迫使在线服务的部署团队不得不将跟踪系统关停。
2. **应用透明**，也就是**代码的侵入性小**
    **即也作为业务组件，应当尽可能少入侵或者无入侵其他业务系统，对于使用方透明，减少开发人员的负担**。
    对于应用的程序员来说，是不需要知道有跟踪系统这回事的。如果一个跟踪系统想生效，就必须需要依赖应用的开发者主动配合，那么这个跟踪系统也太脆弱了，往往由于跟踪系统在应用中植入代码的bug或疏忽导致应用出问题，这样才是无法满足对跟踪系统“无所不在的部署”这个需求。
3. **可扩展性**
    **一个优秀的调用跟踪系统必须支持分布式部署，具备良好的可扩展性。能够支持的组件越多当然越好**。或者提供便捷的插件开发API，对于一些没有监控到的组件，应用开发者也可以自行扩展。
4. **数据的分析**
    **数据的分析要快 ，分析的维度尽可能多**。跟踪系统能提供足够快的信息反馈，就可以对生产环境下的异常状况做出快速反应。**分析的全面，能够避免二次开发**。


### 1.3 Dapper的分布式跟踪原理

先来看一次请求调用示例：

1. 包括：前端（A），两个中间层（B和C），以及两个后端（D和E）
2. 当用户发起一个请求时，首先到达前端A服务，然后分别对B服务和C服务进行RPC调用；
3. B服务处理完给A做出响应，但是C服务还需要和后端的D服务和E服务交互之后再返还给A服务，最后由A服务来响应用户的请求；


![Dapper的分布式跟踪](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535337727679.png)

Dapper是如何来跟踪记录这次请求呢？

#### 1.3.1  **跟踪树和span**

Span是dapper的**基本工作单元**，一次链路调用（可以是RPC，DB等没有特定的限制）创建一个span，通过一个64位ID标识它；同时附加（Annotation）作为payload负载信息，用于记录性能等数据。

![5个span在Dapper跟踪树种短暂的关联关系](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535338174615.png)

上图说明了span在一次大的跟踪过程中是什么样的。**Dapper记录了span名称，以及每个span的ID和父ID，以重建在一次追踪过程中不同span之间的关系**。如果一个span没有父ID被称为root span。**所有span都挂在一个特定的跟踪上，也共用一个跟踪id**。

再来看下**Span的细节**：

![一个单独的span的细节图](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535338251754.png)

**Span数据结构**：


    type Span struct {
        TraceID    int64 // 用于标示一次完整的请求id
        Name       string
        ID         int64 // 当前这次调用span_id
        ParentID   int64 // 上层服务的调用span_id  最上层服务parent_id为null
        Annotation []Annotation // 用于标记的时间戳
        Debug      bool
    }


#### 1.3.2 TraceID

类似于 **树结构的Span集合**，表示一次完整的跟踪，从请求到服务器开始，服务器返回response结束，跟踪每次rpc调用的耗时，存在唯一标识trace\_id。比如：你运行的分布式大数据存储一次Trace就由你的一次请求组成。

![Trace](https://user-gold-cdn.xitu.io/2018/6/4/163c9bee3a9d029a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

每种颜色的note标注了一个span，一条链路通过TraceId唯一标识，Span标识发起的请求信息。**树节点是整个架构的基本单元，而每一个节点又是对span的引用**。节点之间的连线表示的span和它的父span直接的关系。虽然span在日志文件中只是简单的代表span的开始和结束时间，他们在整个树形结构中却是相对独立的。

#### 1.3.3 Annotation

Dapper允许应用程序开发人员在Dapper跟踪的过程中添加额外的信息，以监控更高级别的系统行为，或帮助调试问题。这就是Annotation：

**Annotation**，用来记录请求特定事件相关信息（例如时间），**一个span中会有多个annotation注解描述**。通常包含四个注解信息：


> (1) **cs**：Client Start，表示客户端发起请求  
>  (2) **sr**：Server Receive，表示服务端收到请求  
>  (3) **ss**：Server Send，表示服务端完成处理，并将结果发送给客户端  
>  (4) **cr**：Client Received，表示客户端获取到服务端返回信息


**Annotation数据结构**：


    type Annotation struct {
        Timestamp int64
        Value     string
        Host      Endpoint
        Duration  int32
    }
    


#### 1.3.4 采样率

低损耗的是Dapper的一个关键的设计目标，因为如果这个工具价值未被证实但又对性能有影响的话，你可以理解服务运营人员为什么不愿意部署它。

另外，某些类型的Web服务对植入带来的性能损耗确实非常敏感。

因此，除了把Dapper的收集工作对基本组件的性能损耗限制的尽可能小之外，Dapper支持设置采样率来减少性能损耗，同时支持**可变采样**。

## 2. APM组件选型

市面上的全链路监控理论模型大多都是借鉴Google Dapper论文，重点关注以下三种APM组件：


> 1. **[Zipkin](https://link.juejin.im?target=http%3A%2F%2Fzipkin.io%2F)**：由Twitter公司开源，开放源代码分布式的跟踪系统，用于收集服务的定时数据，以解决微服务架构中的延迟问题，包括：数据的收集、存储、查找和展现。
> 2. **[Pinpoint](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fnaver%2Fpinpoint)**：一款对Java编写的大规模分布式系统的APM工具，由韩国人开源的分布式跟踪组件。
> 3. **[Skywalking](https://link.juejin.im?target=http%3A%2F%2Fskywalking.org%2F)**：国产的优秀APM组件，是一个对JAVA分布式应用程序集群的业务运行情况进行追踪、告警和分析的系统。


### 2.1 对比项

主要对比项：

1. **探针的性能**
    主要是agent对服务的吞吐量、CPU和内存的影响。微服务的规模和动态性使得数据收集的成本大幅度提高。
2. **collector的可扩展性**
    能够水平扩展以便支持大规模服务器集群。
3. **全面的调用链路数据分析**
    提供代码级别的可见性以便轻松定位失败点和瓶颈。
4. **对于开发透明，容易开关**
    添加新功能而无需修改代码，容易启用或者禁用。
5. **完整的调用链应用拓扑**
    自动检测应用拓扑，帮助你搞清楚应用的架构


### 2.2 探针的性能

比较关注探针的性能，毕竟APM定位还是工具，如果启用了链路监控组建后，直接导致吞吐量降低过半，那也是不能接受的。对skywalking、zipkin、pinpoint进行了压测，并与基线（未使用探针）的情况进行了对比。

选用了一个常见的基于Spring的应用程序，他包含Spring Boot, Spring MVC，redis客户端，mysql。 监控这个应用程序，每个trace，探针会抓取5个span(1 Tomcat, 1 SpringMVC, 2 Jedis, 1 Mysql)。这边基本和 [skywalkingtest](https://link.juejin.im?target=https%3A%2F%2Flink.juejin.im%3Ftarget%3Dhttps%253A%252F%252Fskywalkingtest.github.io%252FAgent-Benchmarks%252FREADME_zh.html) 的测试应用差不多。

模拟了三种并发用户：500，750，1000。使用jmeter测试，每个线程发送30个请求，设置思考时间为10ms。使用的采样率为1，即100%，这边与生产可能有差别。pinpoint默认的采样率为20，即50%，通过设置agent的配置文件改为100%。zipkin默认也是1。组合起来，一共有12种。下面看下汇总表：

![探针性能](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339573733.png)

从上表可以看出，在三种链路监控组件中，**skywalking的探针对吞吐量的影响最小，zipkin的吞吐量居中。pinpoint的探针对吞吐量的影响较为明显**，在500并发用户时，测试服务的吞吐量从1385降低到774，影响很大。然后再看下CPU和memory的影响，在内部服务器进行的压测，对CPU和memory的影响都差不多在10%之内。

### 2.3 collector的可扩展性

collector的可扩展性，使得能够水平扩展以便支持大规模服务器集群。

1. **zipkin**
    开发zipkin-Server（其实就是提供的开箱即用包），zipkin-agent与zipkin-Server通过http或者mq进行通信，**http通信会对正常的访问造成影响，所以还是推荐基于mq异步方式通信**，zipkin-Server通过订阅具体的topic进行消费。这个当然是可以扩展的，**多个zipkin-Server实例进行异步消费mq中的监控信息**。
2. **skywalking**
    skywalking的collector支持两种部署方式：**单机和集群模式。collector与agent之间的通信使用了gRPC**。
3. **pinpoint**
    同样，pinpoint也是支持集群和单机部署的。**pinpoint agent通过thrift通信框架，发送链路信息到collector**。


### 2.4 全面的调用链路数据分析

全面的调用链路数据分析，提供代码级别的可见性以便轻松定位失败点和瓶颈。

**zipkin**

![zipkin](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339676013.png)  
**zipkin的链路监控粒度相对没有那么细**，从上图可以看到调用链中具体到接口级别，再进一步的调用信息并未涉及。

**skywalking**

![skywalking](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339690665.png)


    **skywalking 还支持20+的中间件、框架、类库**，比如：主流的dubbo、Okhttp，还有DB和消息中间件。上图skywalking链路调用分析截取的比较简单，网关调用user服务，**由于支持众多的中间件，所以skywalking链路调用分析比zipkin完备些**。


**pinpoint**

![pinpoint](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339702378.png)  
 pinpoint应该是这三种APM组件中，**数据分析最为完备的组件**。提供代码级别的可见性以便轻松定位失败点和瓶颈，上图可以看到对于执行的sql语句，都进行了记录。还可以配置报警规则等，设置每个应用对应的负责人，根据配置的规则报警，支持的中间件和框架也比较完备。

### 2.5  对于开发透明，容易开关

对于开发透明，容易开关，添加新功能而无需修改代码，容易启用或者禁用。我们期望功能可以不修改代码就工作并希望得到代码级别的可见性。

对于这一点，**Zipkin 使用修改过的类库和它自己的容器(Finagle)来提供分布式事务跟踪的功能**。但是，它要求在需要时修改代码。**skywalking和pinpoint都是基于字节码增强的方式，开发人员不需要修改代码，并且可以收集到更多精确的数据因为有字节码中的更多信息**。

### 2.6  完整的调用链应用拓扑

自动检测应用拓扑，帮助你搞清楚应用的架构。

zipkin链路拓扑：  
![zipkin链路拓扑](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339925728.png)

skywalking链路拓扑：

![skywalking链路拓扑](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339959895.png)

![pinpoint](https://gitee.com/jadepeng/blogpic/raw/master/pic/27/1535339973274.png)

三个组件都能实现完整的调用链应用拓扑。相对来说：

- pinpoint界面显示的更加丰富，具体到调用的DB名
- zipkin的拓扑局限于服务于服务之间


### 2.7  社区支持

Zipkin 由 Twitter 开发，可以算得上是明星团队，而 pinpoint的Naver 的团队只是一个默默无闻的小团队，skywalking是国内的明星项目，目前属于apache孵化项目，社区活跃。

### 2.8  总结


|  | zipkin | pinpoint | skywalking |
| --- | --- | --- | --- |
| 探针性能 | 中 | 低 | **高** |
| collector扩展性 | **高** | 中 | **高** |
| 调用链路数据分析 | 低 | **高** | 中 |
| 对开发透明性 | 中 | **高** | **高** |
| 调用链应用拓扑 | 中 | **高** | 中 |
| 社区支持 | **高** | 中 | **高** |


相对来说，skywalking更占优，因此团队采用skywalking作为APM工具。

## 3. 参考内容

本文主要内容参考下文：  
[https://juejin.im/post/5a7a9e0af265da4e914b46f1](https://juejin.im/post/5a7a9e0af265da4e914b46f1)  
[http://bigbully.github.io/Dapper-translation/](http://bigbully.github.io/Dapper-translation/)

