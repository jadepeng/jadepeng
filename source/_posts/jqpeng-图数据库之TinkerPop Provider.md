---
title: 图数据库之TinkerPop Provider
tags: ["图数据库","知识图谱","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-08-13 16:57
---
文章作者:jqpeng
原文链接: [图数据库之TinkerPop Provider](https://www.cnblogs.com/xiaoqi/p/tinkerPop-Provider.html)

Apache TinkerPop 提供了图数据库的抽象接口，方便第三方实现自己的图数据库以接入TinkerPop 技术栈，享受TinkerPop 的Gremlin、算法等福利。TinkerPop将这些第三方称为“Provider ”，知名的Provider包含janusGraph、neo4j、hugegraph等。

Provider包含：

- Graph System Provider

    - Graph Database Provider
    - Graph Processor Provider
- Graph Driver Provider
- Graph Language Provider
- Graph Plugin Provider


## Graph Structure API（图谱数据结构）

Graph最高层的抽象数据结构包含 Graph（图）, Vertex（顶点）, Edge（边）, VertexProperty（属性） and Property.

基于这些基础数据结构，就可以对进行基本的图谱操作。


    Graph graph = TinkerGraph.open(); //1
    Vertex marko = graph.addVertex(T.label, "person", T.id, 1, "name", "marko", "age", 29); //2
    Vertex vadas = graph.addVertex(T.label, "person", T.id, 2, "name", "vadas", "age", 27);
    Vertex lop = graph.addVertex(T.label, "software", T.id, 3, "name", "lop", "lang", "java");
    Vertex josh = graph.addVertex(T.label, "person", T.id, 4, "name", "josh", "age", 32);
    Vertex ripple = graph.addVertex(T.label, "software", T.id, 5, "name", "ripple", "lang", "java");
    Vertex peter = graph.addVertex(T.label, "person", T.id, 6, "name", "peter", "age", 35);
    marko.addEdge("knows", vadas, T.id, 7, "weight", 0.5f); //3
    marko.addEdge("knows", josh, T.id, 8, "weight", 1.0f);
    marko.addEdge("created", lop, T.id, 9, "weight", 0.4f);
    josh.addEdge("created", ripple, T.id, 10, "weight", 1.0f);
    josh.addEdge("created", lop, T.id, 11, "weight", 0.4f);
    peter.addEdge("created", lop, T.id, 12, "weight", 0.2f);


1. 创建一个基于内存存储的TinkerGraph 实例（TinkerGraph是官方实现的，基于内存的Graph）


2 .创建一个顶点

1. 创建边


上面的代码构建了一个基本的图，下面的代码演示如何进行图谱的操作。

![图谱操作](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/13/1597305698157.png)

## 实现 Gremlin-Core

一个标准的Graph Provider需要实现OLTP 和OLAP两类接口，官方推荐学习TinkerGraph(in-memory OLTP and OLAP in tinkergraph-gremlin)，以及 Neo4jGraph (OLTP w/ transactions in neo4j-gremlin) ，还有  
 Neo4jGraph (OLTP w/ transactions in neo4j-gremlin) ，还有 HadoopGraph (OLAP in hadoop-gremlin) 。

1. 在线事务处理 Graph Systems (**OLTP**)



    1.  数据结构 API: `Graph`, `Element`, `Vertex`, `Edge`, `Property` and `Transaction` (if transactions are supported).
    
    2.  处理API : `TraversalStrategy` instances for optimizing Gremlin traversals to the provider’s graph system (i.e. `TinkerGraphStepStrategy`).


1. 在线分析 图系统 (**OLAP**)

    1. Everything required of OLTP is required of OLAP (but not vice versa).
    2. GraphComputer API: `GraphComputer`, `Messenger`, `Memory`.


### OLTP 实现

需要实现structure包下的interface，包含`Graph`, `Vertex`, `Edge`, `Property`, `Transaction`等等。

- `Graph`实现时，需要命名为`XXXGraph` (举例： TinkerGraph, Neo4jGraph, HadoopGraph, etc.).
    - 需要兼容`GraphFactory` ，也就是提供一个静态的 `Graph open(Configuration)` 方法。


### OLAP 实现

需要实现：

1. `GraphComputer`: 图计算器，提供隔离环境，执行VertexProgram,和MapReduce任务.
2. `Memory`: A global blackboard for ANDing, ORing, INCRing, and SETing values for specified keys.
3. `Messenger`: The system that collects and distributes messages being propagated by vertices executing the VertexProgram application.
4. `MapReduce.MapEmitter`: The system that collects key/value pairs being emitted by the MapReduce applications map-phase.
5. `MapReduce.ReduceEmitter`: The system that collects key/value pairs being emitted by the MapReduce applications combine- and reduce-phases.


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


