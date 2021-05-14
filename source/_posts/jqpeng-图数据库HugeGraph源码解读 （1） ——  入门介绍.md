---
title: 图数据库HugeGraph源码解读 （1） ——  入门介绍
tags: ["知识图谱","hugegraph","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-08-10 19:32
---
文章作者:jqpeng
原文链接: [图数据库HugeGraph源码解读 （1） ——  入门介绍](https://www.cnblogs.com/xiaoqi/p/hugegraph-summary.html)

## HugeGraph介绍

以下引自官方文档：


    HugeGraph是一款易用、高效、通用的开源图数据库系统（Graph Database，GitHub项目地址）， 实现了Apache TinkerPop3框架及完全兼容Gremlin查询语言， 具备完善的工具链组件，助力用户轻松构建基于图数据库之上的应用和产品。HugeGraph支持百亿以上的顶点和边快速导入，并提供毫秒级的关联关系查询能力（OLTP）， 并可与Hadoop、Spark等大数据平台集成以进行离线分析（OLAP）。
    
    HugeGraph典型应用场景包括深度关系探索、关联分析、路径搜索、特征抽取、数据聚类、社区检测、 知识图谱等，适用业务领域有如网络安全、电信诈骗、金融风控、广告推荐、社交网络和智能机器人等。


划重点：  
 - 基于TinkerPop3框架，兼容Gremlin查询语言  
 - OLTP（开源） 与 OLAP(商业版）  
 - 常用图应用支持—— 路径搜索、推荐等

## 架构介绍

### 架构图

HugeGraph包括三个层次的功能，分别是存储层、计算层和用户接口层。 HugeGraph支持OLTP和OLAP两种图计算类型

![HugeGraph架构](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/10/1597055013416.png)

### 组件

HugeGraph的主要功能分为HugeCore、ApiServer、HugeGraph-Client、HugeGraph-Loader和HugeGraph-Studio等组件构成，各组件之间的通信关系如下图所示。

![组件](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/10/1597055066287.png)

其中核心组件：

- HugeCore ：HugeGraph的核心模块，TinkerPop的接口主要在该模块中实现。
- ApiServer ：提供RESTFul Api接口，对外提供Graph Api、Schema Api和Gremlin Api等接口服务。
- HugeGraph-Client：基于Java客户端驱动程序


生态组件：

- HugeGraph-Loader：数据导入模块。HugeGraph-Loader可以扫描并分析现有数据，自动生成Graph Schema创建语言，通过批量方式快速导入数据。
- HugeGraph-Studio：基于Web的可视化IDE环境。以Notebook方式记录Gremlin查询，可视化展示Graph的关联关系。HugeGraph-Studio也是本系统推荐的工具。


`HugeGraph-Studio`  看起来已经被抛弃了，研发团队正开发一个名为'hugegraph-hubble' 的新项目：


    hugegraph-hubble is a graph management and analysis platform that provides features: graph data load, schema management, graph relationship analysis and graphical display.


根据官方的说明，`hubble`定义为图谱管理和分析平台，提供图谱数据加载、schema管理、图分析和可视化展示，目前正在研发中，预计2020年9月份会发布首个版本。

### 设计理念

常见的图数据表示模型有两种：

- RDF（Resource Description Framework）模型： 学术界的选择，通过sparql来进行查询，`jena`，`gStore`等等
- 属性图（Property Graph）模型，工业界的选择，`neo4j`和`janusgraph`都是这种方案。


RDF是W3C标准，而Property Graph是工业标准，受到广大图数据库厂商的广泛支持。HugeGraph采用Property Graph，遵循工业标准。

HugeGraph存储概念模型详见下图：

![HugeGraph概念模型](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/10/1597055603962.png)

主要包含几个部分：

- Vertex（顶点），对应一个实体（Entity）
- Vertex Label(顶点的类型），对应一个概念（Concept）
- 属性（图里的name、age），PropertyKey
- Edge边（图里的lives），对应RDF里的Relation


### 可扩展性

HugeGraph提供了丰富的插件扩展机制，包含几个维度的扩展项：

- 后端存储
- 序列化器
- 自定义配置项
- 分词器


插件实现机制

1. HugeGraph提供插件接口HugeGraphPlugin，通过Java SPI机制支持插件化
2. HugeGraph提供了4个扩展项注册函数：`registerOptions()`、`registerBackend()`、`registerSerializer()`、`registerAnalyzer()`
3. 插件实现者实现相应的Options、Backend、Serializer或Analyzer的接口
4. 插件实现者实现HugeGraphPlugin接口的`register()`方法，在该方法中注册上述第3点所列的具体实现类，并打成jar包
5. 插件使用者将jar包放在HugeGraph Server安装目录的`plugins`目录下，修改相关配置项为插件自定义值，重启即可生效


### 从案例深入源码

想要深入的理解一个系统的源码，先从具体的应用入手。先查看example代码：

`https://github.com/hugegraph/hugegraph/blob/master/hugegraph-example/src/main/java/com/baidu/hugegraph/example/Example1.java`


     public static void main(String[] args) throws Exception {
            LOG.info("Example1 start!");
    
            HugeGraph graph = ExampleUtil.loadGraph();
    
            Example1.showFeatures(graph);
    
            Example1.loadSchema(graph);
            Example1.loadData(graph);
            Example1.testQuery(graph);
            Example1.testRemove(graph);
            Example1.testVariables(graph);
            Example1.testLeftIndexProcess(graph);
    
            Example1.thread(graph);
    
            graph.close();
    
            HugeFactory.shutdown(30L);
        }
    


#### 1. loadGraph

要使用hugegraph，需要先初始化一个`HugeGraph`对象，而`LoadGraph` 正是做这个的。

![loadGraph](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/10/1597056461542.png)


    public static HugeGraph loadGraph(boolean needClear, boolean needProfile) {
            if (needProfile) {
                profile();
            }
    
            registerPlugins();
    
            String conf = "hugegraph.properties";
            try {
                String path = ExampleUtil.class.getClassLoader()
                                         .getResource(conf).getPath();
                File file = new File(path);
                if (file.exists() && file.isFile()) {
                    conf = path;
                }
            } catch (Exception ignored) {
            }
    
            HugeGraph graph = HugeFactory.open(conf);
    
            if (needClear) {
                graph.clearBackend();
            }
            graph.initBackend();
    
            return graph;
        }


##### 1.1 registerPlugins

其中 `registerPlugins` 注册插件，注意上面介绍的扩展机制。hugegraph所有的后端存储都需要通过插件注册。


     public static void registerPlugins() {
            if (registered) {
                return;
            }
            registered = true;
    
            RegisterUtil.registerCassandra();
            RegisterUtil.registerScyllaDB();
            RegisterUtil.registerHBase();
            RegisterUtil.registerRocksDB();
            RegisterUtil.registerMysql();
            RegisterUtil.registerPalo();
        }


注册主要是register配置、序列化器和backend，比如下面是mysql的。


    public static void registerMysql() {
            // Register config
            OptionSpace.register("mysql",
                    "com.baidu.hugegraph.backend.store.mysql.MysqlOptions");
            // Register serializer
            SerializerFactory.register("mysql",
                    "com.baidu.hugegraph.backend.store.mysql.MysqlSerializer");
            // Register backend
            BackendProviderFactory.register("mysql",
                    "com.baidu.hugegraph.backend.store.mysql.MysqlStoreProvider");
        }


##### 1.2 HugeFactory.open

HugeFactory 是Hugraph的工厂类，支持传入Configuraion配置信息，构建一个HugeGraph实例，注意这里为了线程安全，签名采用`synchronized`


     public static synchronized HugeGraph open(Configuration config) {
            HugeConfig conf = config instanceof HugeConfig ?
                              (HugeConfig) config : new HugeConfig(config);
            String name = conf.get(CoreOptions.STORE);
            checkGraphName(name, "graph config(like hugegraph.properties)");
            name = name.toLowerCase();
            HugeGraph graph = graphs.get(name);
            if (graph == null || graph.closed()) {
                graph = new StandardHugeGraph(conf);
                graphs.put(name, graph);
            } else {
                String backend = conf.get(CoreOptions.BACKEND);
                E.checkState(backend.equalsIgnoreCase(graph.backend()),
                             "Graph name '%s' has been used by backend '%s'",
                             name, graph.backend());
            }
            return graph;
        }


这里顺带提下配置文件，通过代码看到，默认是读取`hugegraph.properties`.

##### 1.3 HugeGraph 对象

HugeGraph是一个interface，继承gremlin的Graph接口，定义了图谱的Schema定义、数据存储、查询等API方法。从上面1.2可以看到，默认的实现是`StandardHugeGraph`。


    public interface HugeGraph extends Graph {
    
        public HugeGraph hugegraph();
    
        public SchemaManager schema();
    
        public Id getNextId(HugeType type);
    
        public void addPropertyKey(PropertyKey key);
        public void removePropertyKey(Id key);
        public Collection<PropertyKey> propertyKeys();
        public PropertyKey propertyKey(String key);
        public PropertyKey propertyKey(Id key);
        public boolean existsPropertyKey(String key);
    
    ...
       
    


##### 1.4 graph.clearBackend 与initBackend

clearBackend将后端数据清理，initBackend初始化基本的数据结构。

#### 2. loadSchema

该方法，用来定义schema：


    public static void loadSchema(final HugeGraph graph) {
    
            SchemaManager schema = graph.schema();
    
            // Schema changes will be commit directly into the back-end
            LOG.info("===============  propertyKey  ================");
            schema.propertyKey("id").asInt().create();
            schema.propertyKey("name").asText().create();
            schema.propertyKey("gender").asText().create();
            schema.propertyKey("instructions").asText().create();
            schema.propertyKey("category").asText().create();
            schema.propertyKey("year").asInt().create();
            schema.propertyKey("time").asText().create();
            schema.propertyKey("timestamp").asDate().create();
            schema.propertyKey("ISBN").asText().create();
            schema.propertyKey("calories").asInt().create();
            schema.propertyKey("amount").asText().create();
            schema.propertyKey("stars").asInt().create();
            schema.propertyKey("age").asInt().valueSingle().create();
            schema.propertyKey("comment").asText().valueSet().create();
            schema.propertyKey("contribution").asText().valueSet().create();
            schema.propertyKey("nickname").asText().valueList().create();
            schema.propertyKey("lived").asText().create();
            schema.propertyKey("country").asText().valueSet().create();
            schema.propertyKey("city").asText().create();
            schema.propertyKey("sensor_id").asUUID().create();
            schema.propertyKey("versions").asInt().valueList().create();
    
            LOG.info("===============  vertexLabel  ================");
    
            schema.vertexLabel("person")
                  .properties("name", "age", "city")
                  .primaryKeys("name")
                  .create();
            schema.vertexLabel("author")
                  .properties("id", "name", "age", "lived")
                  .primaryKeys("id").create();
            schema.vertexLabel("language").properties("name", "versions")
                  .primaryKeys("name").create();
            schema.vertexLabel("recipe").properties("name", "instructions")
                  .primaryKeys("name").create();
            schema.vertexLabel("book").properties("name")
                  .primaryKeys("name").create();
            schema.vertexLabel("reviewer").properties("name", "timestamp")
                  .primaryKeys("name").create();
    
            // vertex label must have the properties that specified in primary key
            schema.vertexLabel("FridgeSensor").properties("city")
                  .primaryKeys("city").create();
    
            LOG.info("===============  vertexLabel & index  ================");
            schema.indexLabel("personByCity")
                  .onV("person").secondary().by("city").create();
            schema.indexLabel("personByAge")
                  .onV("person").range().by("age").create();
    
            schema.indexLabel("authorByLived")
                  .onV("author").search().by("lived").create();
    
            // schemaManager.getVertexLabel("author").index("byName").secondary().by("name").add();
            // schemaManager.getVertexLabel("recipe").index("byRecipe").materialized().by("name").add();
            // schemaManager.getVertexLabel("meal").index("byMeal").materialized().by("name").add();
            // schemaManager.getVertexLabel("ingredient").index("byIngredient").materialized().by("name").add();
            // schemaManager.getVertexLabel("reviewer").index("byReviewer").materialized().by("name").add();
    
            LOG.info("===============  edgeLabel  ================");
    
            schema.edgeLabel("authored").singleTime()
                  .sourceLabel("author").targetLabel("book")
                  .properties("contribution", "comment")
                  .nullableKeys("comment")
                  .create();
    
            schema.edgeLabel("write").multiTimes().properties("time")
                  .sourceLabel("author").targetLabel("book")
                  .sortKeys("time")
                  .create();
    
            schema.edgeLabel("look").multiTimes().properties("timestamp")
                  .sourceLabel("person").targetLabel("book")
                  .sortKeys("timestamp")
                  .create();
    
            schema.edgeLabel("created").singleTime()
                  .sourceLabel("author").targetLabel("language")
                  .create();
    
            schema.edgeLabel("rated")
                  .sourceLabel("reviewer").targetLabel("recipe")
                  .create();
        }


划重点：  
 - SchemaManager schema = graph.schema() 获取SchemaManager  
 - schema.propertyKey(NAME).asXXType().create() 创建属性  
 - schema.vertexLabel("person") // 定义概念  
 .properties("name", "age", "city")  // 定义概念的属性  
 .primaryKeys("name")  // 定义primary Keys，primary Key组合后可以唯一确定一个实体  
 .create();  
 -  schema.indexLabel("personByCity").onV("person").secondary().by("city").create();  定义索引  
 -  schema.edgeLabel("authored").singleTime()  
 .sourceLabel("author").targetLabel("book")  
 .properties("contribution", "comment")  
 .nullableKeys("comment")  
 .create();  // 定义关系

#### 3. loadData

创建实体，注意格式，K-V成对出现：


    graph.addVertex(T.label, "book", "name", "java-3");


创建关系，Vertex的addEdge方法：


        Vertex james = tx.addVertex(T.label, "author", "id", 1,
                                    "name", "James Gosling",  "age", 62,
                                    "lived", "San Francisco Bay Area");
    
        Vertex java = tx.addVertex(T.label, "language", "name", "java",
                                   "versions", Arrays.asList(6, 7, 8));
        Vertex book1 = tx.addVertex(T.label, "book", "name", "java-1");
        Vertex book2 = tx.addVertex(T.label, "book", "name", "java-2");
        Vertex book3 = tx.addVertex(T.label, "book", "name", "java-3");
    
        james.addEdge("created", java);
        james.addEdge("authored", book1,
                      "contribution", "1990-1-1",
                      "comment", "it's a good book",
                      "comment", "it's a good book",
                      "comment", "it's a good book too");
        james.addEdge("authored", book2, "contribution", "2017-4-28");
    
        james.addEdge("write", book2, "time", "2017-4-28");
        james.addEdge("write", book3, "time", "2016-1-1");
        james.addEdge("write", book3, "time", "2017-4-28");	


添加后，需要commit

#### 4. testQuery 测试查询

查询主要通过`GraphTraversal`, 可以通过`graph.traversal()`获得：


    public static void testQuery(final HugeGraph graph) {
            // query all
            GraphTraversal<Vertex, Vertex> vertices = graph.traversal().V();
            int size = vertices.toList().size();
            assert size == 12;
            System.out.println(">>>> query all vertices: size=" + size);
    
            // query by label
            vertices = graph.traversal().V().hasLabel("person");
            size = vertices.toList().size();
            assert size == 5;
            System.out.println(">>>> query all persons: size=" + size);
    
            // query vertex by primary-values
            vertices = graph.traversal().V().hasLabel("author").has("id", 1);
            List<Vertex> vertexList = vertices.toList();
            assert vertexList.size() == 1;
            System.out.println(">>>> query vertices by primary-values: " +
                               vertexList);
    
            VertexLabel author = graph.schema().getVertexLabel("author");
            String authorId = String.format("%s:%s", author.id().asString(), "11");
    
            // query vertex by id and query out edges
            vertices = graph.traversal().V(authorId);
            GraphTraversal<Vertex, Edge> edgesOfVertex = vertices.outE("created");
            List<Edge> edgeList = edgesOfVertex.toList();
            assert edgeList.size() == 1;
            System.out.println(">>>> query edges of vertex: " + edgeList);
    
            vertices = graph.traversal().V(authorId);
            vertexList = vertices.out("created").toList();
            assert vertexList.size() == 1;
            System.out.println(">>>> query vertices of vertex: " + vertexList);
    
            // query edge by sort-values
            vertices = graph.traversal().V(authorId);
            edgesOfVertex = vertices.outE("write").has("time", "2017-4-28");
            edgeList = edgesOfVertex.toList();
            assert edgeList.size() == 2;
            System.out.println(">>>> query edges of vertex by sort-values: " +
                               edgeList);
    
            // query vertex by condition (filter by property name)
            ConditionQuery q = new ConditionQuery(HugeType.VERTEX);
            PropertyKey age = graph.propertyKey("age");
            q.key(HugeKeys.PROPERTIES, age.id());
            if (graph.backendStoreFeatures()
                     .supportsQueryWithContainsKey()) {
                Iterator<Vertex> iter = graph.vertices(q);
                assert iter.hasNext();
                System.out.println(">>>> queryVertices(age): " + iter.hasNext());
                while (iter.hasNext()) {
                    System.out.println(">>>> queryVertices(age): " + iter.next());
                }
            }
    
            // query all edges
            GraphTraversal<Edge, Edge> edges = graph.traversal().E().limit(2);
            size = edges.toList().size();
            assert size == 2;
            System.out.println(">>>> query all edges with limit 2: size=" + size);
    
            // query edge by id
            EdgeLabel authored = graph.edgeLabel("authored");
            VertexLabel book = graph.schema().getVertexLabel("book");
            String book1Id = String.format("%s:%s", book.id().asString(), "java-1");
            String book2Id = String.format("%s:%s", book.id().asString(), "java-2");
    
            String edgeId = String.format("S%s>%s>%s>S%s",
                                          authorId, authored.id(), "", book2Id);
            edges = graph.traversal().E(edgeId);
            edgeList = edges.toList();
            assert edgeList.size() == 1;
            System.out.println(">>>> query edge by id: " + edgeList);
    
            Edge edge = edgeList.get(0);
            edges = graph.traversal().E(edge.id());
            edgeList = edges.toList();
            assert edgeList.size() == 1;
            System.out.println(">>>> query edge by id: " + edgeList);
    
            // query edge by condition
            q = new ConditionQuery(HugeType.EDGE);
            q.eq(HugeKeys.OWNER_VERTEX, IdGenerator.of(authorId));
            q.eq(HugeKeys.DIRECTION, Directions.OUT);
            q.eq(HugeKeys.LABEL, authored.id());
            q.eq(HugeKeys.SORT_VALUES, "");
            q.eq(HugeKeys.OTHER_VERTEX, IdGenerator.of(book1Id));
    
            Iterator<Edge> edges2 = graph.edges(q);
            assert edges2.hasNext();
            System.out.println(">>>> queryEdges(id-condition): " +
                               edges2.hasNext());
            while (edges2.hasNext()) {
                System.out.println(">>>> queryEdges(id-condition): " +
                                   edges2.next());
            }
    
            // NOTE: query edge by has-key just supported by Cassandra
            if (graph.backendStoreFeatures().supportsQueryWithContainsKey()) {
                PropertyKey contribution = graph.propertyKey("contribution");
                q.key(HugeKeys.PROPERTIES, contribution.id());
                Iterator<Edge> edges3 = graph.edges(q);
                assert edges3.hasNext();
                System.out.println(">>>> queryEdges(contribution): " +
                                   edges3.hasNext());
                while (edges3.hasNext()) {
                    System.out.println(">>>> queryEdges(contribution): " +
                                       edges3.next());
                }
            }
    
            // query by vertex label
            vertices = graph.traversal().V().hasLabel("book");
            size = vertices.toList().size();
            assert size == 5;
            System.out.println(">>>> query all books: size=" + size);
    
            // query by vertex label and key-name
            vertices = graph.traversal().V().hasLabel("person").has("age");
            size = vertices.toList().size();
            assert size == 5;
            System.out.println(">>>> query all persons with age: size=" + size);
    
            // query by vertex props
            vertices = graph.traversal().V().hasLabel("person")
                            .has("city", "Taipei");
            vertexList = vertices.toList();
            assert vertexList.size() == 1;
            System.out.println(">>>> query all persons in Taipei: " + vertexList);
    
            vertices = graph.traversal().V().hasLabel("person").has("age", 19);
            vertexList = vertices.toList();
            assert vertexList.size() == 1;
            System.out.println(">>>> query all persons age==19: " + vertexList);
    
            vertices = graph.traversal().V().hasLabel("person")
                            .has("age", P.lt(19));
            vertexList = vertices.toList();
            assert vertexList.size() == 1;
            assert vertexList.get(0).property("age").value().equals(3);
            System.out.println(">>>> query all persons age<19: " + vertexList);
    
            String addr = "Bay Area";
            vertices = graph.traversal().V().hasLabel("author")
                            .has("lived", Text.contains(addr));
            vertexList = vertices.toList();
            assert vertexList.size() == 1;
            System.out.println(String.format(">>>> query all authors lived %s: %s",
                               addr, vertexList));
        }


划重点

##### 查询指定label的实体：


     vertices = graph.traversal().V().hasLabel("person");
     size = vertices.toList().size();


##### 根据primary-values查询实体：


     vertices = graph.traversal().V().hasLabel("author").has("id", 1);
            List<Vertex> vertexList = vertices.toList();


##### 查询edge：

查询所有edge：


    GraphTraversal<Edge, Edge> edges = graph.traversal().E().limit(2);


根据ID查询edge：


    	EdgeLabel authored = graph.edgeLabel("authored");
        VertexLabel book = graph.schema().getVertexLabel("book");
        String book1Id = String.format("%s:%s", book.id().asString(), "java-1");
        String book2Id = String.format("%s:%s", book.id().asString(), "java-2");
    
        String edgeId = String.format("S%s>%s>%s>S%s",
                                      authorId, authored.id(), "", book2Id);
        edges = graph.traversal().E(edgeId);


注意，edge的id由几个字段拼接起来的：  "S%s&gt;%s&gt;%s&gt;S%s",authorId, authored.id(), "", book2Id)

根据条件查询edge：


     q = new ConditionQuery(HugeType.EDGE);
            q.eq(HugeKeys.OWNER_VERTEX, IdGenerator.of(authorId));
            q.eq(HugeKeys.DIRECTION, Directions.OUT);
            q.eq(HugeKeys.LABEL, authored.id());
            q.eq(HugeKeys.SORT_VALUES, "");
            q.eq(HugeKeys.OTHER_VERTEX, IdGenerator.of(book1Id));
    
            Iterator<Edge> edges2 = graph.edges(q);
            assert edges2.hasNext();
            System.out.println(">>>> queryEdges(id-condition): " +
                               edges2.hasNext());
            while (edges2.hasNext()) {
                System.out.println(">>>> queryEdges(id-condition): " +
                                   edges2.next());
            }


可以指定DIRECTION，

#### 5. 删除

删除Vetex，调用vetex自带的remove方法


           // remove vertex (and its edges)
            List<Vertex> vertices = graph.traversal().V().hasLabel("person")
                                         .has("age", 19).toList();
            assert vertices.size() == 1;
            Vertex james = vertices.get(0);
            Vertex book6 = graph.addVertex(T.label, "book", "name", "java-6");
            james.addEdge("look", book6, "timestamp", "2017-5-2 12:00:08.0");
            james.addEdge("look", book6, "timestamp", "2017-5-3 12:00:08.0");
            graph.tx().commit();
            assert graph.traversal().V(book6.id()).bothE().hasNext();
            System.out.println(">>>> removing vertex: " + james);
            james.remove();
            graph.tx().commit();
            assert !graph.traversal().V(james.id()).hasNext();
            assert !graph.traversal().V(book6.id()).bothE().hasNext();
    
        


删除关系，也类似：


        // remove edge
            VertexLabel author = graph.schema().getVertexLabel("author");
            String authorId = String.format("%s:%s", author.id().asString(), "11");
            EdgeLabel authored = graph.edgeLabel("authored");
            VertexLabel book = graph.schema().getVertexLabel("book");
            String book2Id = String.format("%s:%s", book.id().asString(), "java-2");
    
            String edgeId = String.format("S%s>%s>%s>S%s",
                                          authorId, authored.id(), "", book2Id);
    
            List <Edge> edges = graph.traversal().E(edgeId).toList();
            assert edges.size() == 1;
            Edge edge = edges.get(0);
            System.out.println(">>>> removing edge: " + edge);
            edge.remove();
            graph.tx().commit();
            assert !graph.traversal().E(edgeId).hasNext();
    


## 小结

本文初步介绍了hugegraph设计理念、基本使用等。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


