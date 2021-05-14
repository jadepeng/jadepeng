---
title: sparql  查询语句快速入门
tags: ["sparql","知识图谱","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-08-23 15:00
---
文章作者:jqpeng
原文链接: [sparql  查询语句快速入门](https://www.cnblogs.com/xiaoqi/p/sparql.html)

## 介绍


> RDF is a directed, labeled graph data format for representing information in the Web. RDF is often used to represent, among other things, personal information, social networks, metadata about digital artifacts, as well as to provide a means of integration over disparate sources of information. This specification defines the syntax and semantics of the SPARQL query language for RDF.  
>  The SPARQL query language for RDF is designed to meet the use cases and requirements identified by the RDF Data Access Working Group in RDF Data Access Use Cases and Requirements [UCNR].


SPARQL即SPARQL Protocol and RDF Query Language的递归缩写，被专门设计用来访问和操作RDF数据，是语义网的核心技术之一。W3C的RDF数据存取小组（RDF Data Access Working Group, RDAWG）对其进行了标准化。2008年1月15日，SPARQL正式成为一项W3C推荐标准。

我们可以将抽取的RDF三元组导入Apache Jena Fuseki，通过SPARQL进行查询：

![SParql demo](https://gitee.com/jadepeng/pic/raw/master/pic/2019/8/23/1566543446161.png)

## 简单查询


| SQL | sparql |
| --- | --- |
| SELECT title from book where id='book1' | SELECT ?title    <br>WHERE   <br>{ &lt;[http://example.org/book/book1](http://example.org/book/book1)&gt; &lt;[http://purl.org/dc/elements/1.1/title](http://purl.org/dc/elements/1.1/title)&gt; ?title .   <br> } |


Query Result:


| title |
| --- |
| "SPARQL Tutorial" |


## 多字段匹配

RDF 数据


    @prefix foaf:  <http://xmlns.com/foaf/0.1/> .
    
    _:a  foaf:name   "Johnny Lee Outlaw" .
    _:a  foaf:mbox   <mailto:jlow@example.com> .
    _:b  foaf:name   "Peter Goodguy" .
    _:b  foaf:mbox   <mailto:peter@example.org> .
    _:c  foaf:mbox   <mailto:carol@example.org> .


sparql:


    PREFIX foaf:   <http://xmlns.com/foaf/0.1/>
    SELECT ?name ?mbox
    WHERE
      { ?x foaf:name ?name .
        ?x foaf:mbox ?mbox }


SQL:


    SELECT ?name ?mbox
    from foaf


查询结果:


| name | mbox |
| --- | --- |
| "Johnny Lee Outlaw" | [mailto:jlow@example.com](mailto:jlow@example.com) |
| "Peter Goodguy" | [mailto:peter@example.org](mailto:peter@example.org) |


## 数据属性匹配

对于string类型，需要用双引号包裹起来。

sparql:


    SELECT ?v WHERE { ?v ?p "cat" }


SQL:


    SELECT *
    from ns
    where p='cat'


对于数字类型：

sparql:


    SELECT ?v WHERE { ?v ?p 42 }


SQL:


    SELECT * from ns where p= 42


另外，在spaql里可以指定匹配的类型：


    SELECT ?v WHERE { ?v ?p "abc"^^<http://example.org/datatype#specialDatatype> }


## 条件过滤

### 模糊匹配

通过`regex`函数可以进行字符串正则匹配，通过`FILTER`进行过滤


    
    PREFIX  dc:  <http://purl.org/dc/elements/1.1/>
    SELECT  ?title
    WHERE   { ?x dc:title ?title
              FILTER regex(?title, "web", "i" ) 
            }
    


SQL:


    SELECT * from table where title like '%web%'


### 数字比较


    
    PREFIX  dc:  <http://purl.org/dc/elements/1.1/>
    PREFIX  ns:  <http://example.org/ns#>
    SELECT  ?title ?price
    WHERE   { ?x ns:price ?price .
              FILTER (?price < 30.5)
              ?x dc:title ?title . }
    


SQL:


    SELECT title,price from table where price <30.5


## OPTIONAL（可选值）

RDF 数据，用户Bob没有mbox，而用户Alice有两个mbox


    @prefix foaf:       <http://xmlns.com/foaf/0.1/> .
    @prefix rdf:        <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
    
    _:a  rdf:type        foaf:Person .
    _:a  foaf:name       "Alice" .
    _:a  foaf:mbox       <mailto:alice@example.com> .
    _:a  foaf:mbox       <mailto:alice@work.example> .
    
    _:b  rdf:type        foaf:Person .
    _:b  foaf:name       "Bob" .


正常查询，因为Bob没有mbox，所以查询不出来，可以通过`OPTIONAL`标记mbox为可选，这样Bob就可以查询出来。

sparql:


    PREFIX foaf: <http://xmlns.com/foaf/0.1/>
    SELECT ?name ?mbox
    WHERE  { ?x foaf:name  ?name .
             OPTIONAL { ?x  foaf:mbox  ?mbox }
           }


查询结果


| name | mbox |
| --- | --- |
| "Alice" | [mailto:alice@example.com](mailto:alice@example.com) |
| "Alice" | [mailto:alice@work.example](mailto:alice@work.example) |
| "Bob" |  |


可以看到， `"Bob"`的 `mbox`是空值。

对于关系型数据库，可以假设两个表

User { id,name}  
 Mbox {id,uid,name} (uid为外键）

对应的sql：


    SELECT user.name AS name,mbox.name AS mboxName
    FROM User user
    LEFT OUTER JOIN Mbox mbox ON mbox.uid=user.id


## OPTIONAL + FILTER

OPTIONAL 可以和FILTER 组合使用


    PREFIX  dc:  <http://purl.org/dc/elements/1.1/>
    PREFIX  ns:  <http://example.org/ns#>
    SELECT  ?title ?price
    WHERE   { ?x dc:title ?title .
              OPTIONAL { ?x ns:price ?price . FILTER (?price < 30) }
            }        
    


## UNION

Data:


    @prefix dc10:  <http://purl.org/dc/elements/1.0/> .
    @prefix dc11:  <http://purl.org/dc/elements/1.1/> .
    
    _:a  dc10:title     "SPARQL Query Language Tutorial" .
    _:a  dc10:creator   "Alice" .
    
    _:b  dc11:title     "SPARQL Protocol Tutorial" .
    _:b  dc11:creator   "Bob" .
    
    _:c  dc10:title     "SPARQL" .
    _:c  dc11:title     "SPARQL (updated)" 
    


查询:


    PREFIX dc10:  <http://purl.org/dc/elements/1.0/>
    PREFIX dc11:  <http://purl.org/dc/elements/1.1/>
    
    SELECT ?title
    WHERE  { { ?book dc10:title  ?title } UNION { ?book dc11:title  ?title } }
    


Query result:


    title
    "SPARQL Protocol Tutorial"
    "SPARQL"
    "SPARQL (updated)"
    "SPARQL Query Language Tutorial"
    


## 排序

和sql一样，使用ORDER BY 排序，示例如下：


    PREFIX foaf:    <http://xmlns.com/foaf/0.1/>
    
    SELECT ?name
    WHERE { ?x foaf:name ?name ; :empId ?emp }
    ORDER BY ?name DESC(?emp)


## 去重

和sql一样，使用DISTINCT来去重，示例如下：


    PREFIX foaf:    <http://xmlns.com/foaf/0.1/>
    SELECT DISTINCT ?name WHERE { ?x foaf:name ?name }


## 判断是否存在

使用ask来判断是否有解决方案


    PREFIX foaf:    <http://xmlns.com/foaf/0.1/>
    ASK  { ?x foaf:name  "Alice" ;
              foaf:mbox  <mailto:alice@work.example> }


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


