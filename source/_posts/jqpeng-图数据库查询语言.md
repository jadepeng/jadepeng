---
title: 图数据库查询语言
tags: ["知识图谱","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-11-17 14:48
---
文章作者:jqpeng
原文链接: [图数据库查询语言](https://www.cnblogs.com/xiaoqi/p/graph-query-lang.html)

本文介绍图数据库支持的gremlin和Cypher查询语言。

## 初始化数据

可使用gremlin api执行

## gremlin api

POST [http://localhost:8080/gremlin](http://localhost:8080/gremlin)


    {"gremlin":"这里是语句",
     "bindings": {},
    "language": "gremlin-groovy",
    "aliases": {	"graph": "graphname", 	"g": "__g_graphname"}
    }


### schema


    schema = hugegraph.schema()
    
    schema.propertyKey("name").asText().ifNotExist().create()
    schema.propertyKey("age").asInt().ifNotExist().create()
    schema.propertyKey("time").asInt().ifNotExist().create()
    schema.propertyKey("reason").asText().ifNotExist().create()
    schema.propertyKey("type").asText().ifNotExist().create()
    
    schema.vertexLabel("character").properties("name", "age", "type").primaryKeys("name").nullableKeys("age").ifNotExist().create()
    schema.vertexLabel("location").properties("name").primaryKeys("name").ifNotExist().create()
    
    schema.edgeLabel("father").link("character", "character").ifNotExist().create()
    schema.edgeLabel("mother").link("character", "character").ifNotExist().create()
    schema.edgeLabel("battled").link("character", "character").properties("time").ifNotExist().create()
    schema.edgeLabel("lives").link("character", "location").properties("reason").nullableKeys("reason").ifNotExist().create()
    schema.edgeLabel("pet").link("character", "character").ifNotExist().create()
    schema.edgeLabel("brother").link("character", "character").ifNotExist().create()


### 插入数据


    // add vertices
    Vertex saturn = graph.addVertex(T.label, "character", "name", "saturn", "age", 10000, "type", "titan")
    Vertex sky = graph.addVertex(T.label, "location", "name", "sky")
    Vertex sea = graph.addVertex(T.label, "location", "name", "sea")
    Vertex jupiter = graph.addVertex(T.label, "character", "name", "jupiter", "age", 5000, "type", "god")
    Vertex neptune = graph.addVertex(T.label, "character", "name", "neptune", "age", 4500, "type", "god")
    Vertex hercules = graph.addVertex(T.label, "character", "name", "hercules", "age", 30, "type", "demigod")
    Vertex alcmene = graph.addVertex(T.label, "character", "name", "alcmene", "age", 45, "type", "human")
    Vertex pluto = graph.addVertex(T.label, "character", "name", "pluto", "age", 4000, "type", "god")
    Vertex nemean = graph.addVertex(T.label, "character", "name", "nemean", "type", "monster")
    Vertex hydra = graph.addVertex(T.label, "character", "name", "hydra", "type", "monster")
    Vertex cerberus = graph.addVertex(T.label, "character", "name", "cerberus", "type", "monster")
    Vertex tartarus = graph.addVertex(T.label, "location", "name", "tartarus")
    
    // add edges
    jupiter.addEdge("father", saturn)
    jupiter.addEdge("lives", sky, "reason", "loves fresh breezes")
    jupiter.addEdge("brother", neptune)
    jupiter.addEdge("brother", pluto)
    neptune.addEdge("lives", sea, "reason", "loves waves")
    neptune.addEdge("brother", jupiter)
    neptune.addEdge("brother", pluto)
    hercules.addEdge("father", jupiter)
    hercules.addEdge("mother", alcmene)
    hercules.addEdge("battled", nemean, "time", 1)
    hercules.addEdge("battled", hydra, "time", 2)
    hercules.addEdge("battled", cerberus, "time", 12)
    pluto.addEdge("brother", jupiter)
    pluto.addEdge("brother", neptune)
    pluto.addEdge("lives", tartarus, "reason", "no fear of death")
    pluto.addEdge("pet", cerberus)
    cerberus.addEdge("lives", tartarus)


![可视化](https://gitee.com/jadepeng/pic/raw/master/pic/2020/11/17/1605595541783.png)

## 创建索引

创建索引，使用REST API：

POST [http://localhost:8080/graphs/hugegraph/schema/indexlabels](http://localhost:8080/graphs/hugegraph/schema/indexlabels)


    {"name": "characterAge","base_type": "VERTEX_LABEL","base_value": "character","index_type": "RANGE","fields": [	"age"]
    }


## 查询

### API 说明

支持`gremlin`、`sparql`和`Cypher` api，推荐gremlin和Cypher

#### cypher api


    http://127.0.0.1:8080/graphs/hugegraph/cypher?cypher=


eg:


    curl http://127.0.0.1:8080/graphs/hugegraph/cypher?cypher=MATCH%20(n:character)-[:lives]-%3E(location)-[:lives]-(cohabitants)%20where%20n.name=%27pluto%27%20return%20cohabitants.name


#### gremlin api

POST [http://localhost:8080/gremlin](http://localhost:8080/gremlin)


    {"gremlin":"这里是语句",
     "bindings": {},
    "language": "gremlin-groovy",
    "aliases": {	"graph": "graphname", 	"g": "__g_graphname"}
    }


#### sparql api


    GET http://127.0.0.1:8080/graphs/hugegraph/sparql?sparql=SELECT%20*%20WHERE%20{%20}


**1. 查询hercules的祖父**


    g.V().hasLabel('character').has('name','hercules').out('father').out('father')
    


也可以通过`repeat`方式：


    g.V().hasLabel('character').has('name','hercules').repeat(__.out('father')).times(2)
    


**cypher**


    MATCH (n:character)-[:father]->()-[:father]->(grandfather) where n.name='hercules' return grandfather


**2. Find the name of hercules's father**


    g.V().hasLabel('character').has('name','hercules').out('father').value('name')


**cypher**


    MATCH (n:character)-[:father]->(father) where n.name='hercules' return father.name


**3. Find the characters with age &gt; 100**


    g.V().hasLabel('character').has('age',gt(100))
    


**cypher**


    MATCH (n:character) where n.age > 10 return n


**4. Find who are pluto's cohabitants**


    g.V().hasLabel('character').has('name','pluto').out('lives').in('lives').values('name')
    


**cypher**


    MATCH (n:character)-[:lives]->(location)-[:lives]-(cohabitants) where n.name='pluto' return cohabitants.name


**5. Find pluto can't be his own cohabitant**


    pluto = g.V().hasLabel('character').has('name', 'pluto')
    g.V(pluto).out('lives').in('lives').where(is(neq(pluto)).values('name')
    
    // use 'as'
    g.V().hasLabel('character').has('name', 'pluto').as('x').out('lives').in('lives').where(neq('x')).values('name')
    



    cypher> MATCH (src:character{name:"pluto"})-[:lives]->()<-[:lives]-(dst:character) RETURN dst.name
    


**6. Pluto's Brothers**


    pluto = g.V().hasLabel('character').has('name', 'pluto').next()
    // where do pluto's brothers live?
    g.V(pluto).out('brother').out('lives').values('name')
    
    // which brother lives in which place?
    g.V(pluto).out('brother').as('god').out('lives').as('place').select('god','place')
    
    // what is the name of the brother and the name of the place?
    g.V(pluto).out('brother').as('god').out('lives').as('place').select('god','place').by('name')



    MATCH (src:Character{name:"pluto"})-[:brother]->(bro:Character)-[:lives]->(dst)
    RETURN bro.name, dst.name


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


