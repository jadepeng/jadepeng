---
title: hugegraph 支持sparql 与cypher
tags: ["知识图谱","hugegraph","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-11-17 15:10
---
文章作者:jqpeng
原文链接: [hugegraph 支持sparql 与cypher](https://www.cnblogs.com/xiaoqi/p/hugegraph-cypher.html)

hugegraph 是百度开源的基于tinkerpop的图数据库，支持通过gremlin进行查询。

这里，我们来扩展支持sparql 与cypher。

## sparql支持

github上有`SparqlToGremlinCompiler`，可以支持将sparql转GraphTraversal，集成该工具库即可：


    @Path("graphs/{graph}/sparql")
    @Singleton
    public class SparqlAPI extends API {
    
        private static final Logger LOG = Log.logger(SparqlAPI.class);
    
        @GET
        @Timed
        @CompressInterceptor.Compress
        @Produces(APPLICATION_JSON_WITH_CHARSET)
        public String query(@Context GraphManager manager,
                            @PathParam("graph") String graph,
                            @QueryParam("sparql") String sparql) {
            LOG.debug("Graph [{}] query by sparql: {}", graph, sparql);
    
            E.checkArgument(sparql != null && !sparql.isEmpty(),
                    "The sparql parameter can't be null or empty");
    
            HugeGraph g = graph(manager, graph);
            GraphTraversal<Vertex, ?> traversal = SparqlToGremlinCompiler.convertToGremlinTraversal(g, sparql);
            List<?> result = traversal.toList();
            if (result.size() > 0) {
                Object item = result.get(0);
                if (item instanceof Vertex) {
                    return manager.serializer(g).writeVertices((Iterator<Vertex>) result.iterator(), false);
                }
                if (item instanceof Map) {
                    return manager.serializer(g).writeMap((Map) item);
                }
            }
    
            return result.toString();
        }
    }


## cypher 支持

opencypher 提供了`translation`包，支持将cypher转为gremlin：


            <dependency>
                <groupId>org.opencypher.gremlin</groupId>
                <artifactId>translation</artifactId>
                <version>1.0.4</version>
            </dependency>


转换代码：


     TranslationFacade cfog = new TranslationFacade();
     String gremlin = cfog.toGremlinGroovy(cypher);


增加api代码


         @GET
        @Timed
        @CompressInterceptor.Compress
        @Produces(APPLICATION_JSON_WITH_CHARSET)
        public Response query(@Context GraphManager manager,
                              @PathParam("graph") String graph,
                              @Context HugeConfig conf,
                              @Context HttpHeaders headers,
                              @QueryParam("cypher") String cypher) {
            LOG.debug("Graph [{}] query by cypher: {}", graph, cypher);
    
            return getResponse(graph, headers, cypher);
        }
    
        private Response getResponse(@PathParam("graph") String graph, @Context HttpHeaders headers, @QueryParam("cypher") String cypher) {
            E.checkArgument(cypher != null && !cypher.isEmpty(),
                    "The cypher parameter can't be null or empty");
    
            TranslationFacade cfog = new TranslationFacade();
            String gremlin = cfog.toGremlinGroovy(cypher);
            gremlin = "g = " + graph + ".traversal()\n" + gremlin;
            String auth = headers.getHeaderString(HttpHeaders.AUTHORIZATION);
            MultivaluedMap<String, String> params = new MultivaluedHashMap<>();
            params.put("gremlin", Arrays.asList(gremlin));
            Response response = this.client().doGetRequest(auth, params);
            return transformResponseIfNeeded(response);
        }


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


