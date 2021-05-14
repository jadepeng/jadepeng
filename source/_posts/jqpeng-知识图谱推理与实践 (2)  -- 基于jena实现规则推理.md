---
title: 知识图谱推理与实践 (2)  -- 基于jena实现规则推理
tags: ["Jena","知识图谱","推理","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-09-06 17:18
---
文章作者:jqpeng
原文链接: [知识图谱推理与实践 (2)  -- 基于jena实现规则推理](https://www.cnblogs.com/xiaoqi/p/kg-inference-2.html)

本章，介绍 基于`jena`的规则引擎实现推理，并通过两个例子介绍如何coding实现。

## 规则引擎概述

jena包含了一个通用的规则推理机，可以在RDFS和OWL推理机使用，也可以单独使用。

推理机支持在RDF图上推理，提供前向链、后向链和二者混合执行模式。包含RETE engine 和 one tabled datalog engine。可以通过[GenericRuleReasoner](https://jena.apache.org/documentation/javadoc/jena/org/apache/jena/reasoner/rulesys/GenericRuleReasoner.html)来进行配置参数，使用各种推理引擎。要使用 `GenericRuleReasoner`，需要一个规则集来定义其行为.

### Rule的语法与结构

规则通过 [Rule](https://jena.apache.org/documentation/javadoc/jena/org/apache/jena/reasoner/rulesys/Rule.html)对象来进行定义，包含 body terms列表 (premises),head terms列表 (conclusions) 和可选的 name 和可选的direction。

An informal description of the simplified text rule syntax is:


    _Rule_      :=   _bare-rule_ .
              or   [ _bare-rule_ ]
           or   [ ruleName : _bare-rule_ ]
    
    _bare-rule_ :=   _term_, ... _term_ -> _hterm_, ... _hterm_    // forward rule
              or   _bhterm_ <- _term_, ... _term   _ // backward rule
    
    _hterm     :=   term
    _ or   [ _bare-rule_ ]
    
    _term_      :=   (_node_, _node_, _node_)           // triple pattern
              or   (_node_, _node_, _functor_)        // extended triple pattern
              or   builtin(_node_, ... _node_)      // invoke procedural primitive
    
    _bhterm_      :=   (_node_, _node_, _node_)           // triple pattern
    
    _functor_   :=   functorName(_node_, ... _node_)  // structured literal
    
    _node_      :=   _uri-ref_                   // e.g. http://foo.com/eg
              or   prefix:localname          // e.g. rdf:type
              or   <_uri-ref_>          // e.g. <myscheme:myuri>
              or   ?_varname_ // variable
              or   'a literal'                 // a plain string literal
              or   'lex'^^typeURI              // a typed literal, xsd:* type names supported
              or   number                      // e.g. 42 or 25.5
    


逗号 "," 分隔符是可选的.

前向和后向规则语法之间的区别仅与混合执行策略相关，请参见下文。

`_functor_` 是一个扩展的三元组，用于创建和访问文本值。functorName可以是任何简单的标识符。

为保障rules的可读性URI引用支持qname语法。可以使用在 [PrintUtil](https://jena.apache.org/documentation/javadoc/jena/org/apache/jena/util/PrintUtil.html)对象中注册的前缀。

下面是一些规则示例：


    [allID: (?C rdf:type owl:Restriction), (?C owl:onProperty ?P),
         (?C owl:allValuesFrom ?D)  -> (?C owl:equivalentClass all(?P, ?D)) ]
    
    [all2: (?C rdfs:subClassOf all(?P, ?D)) -> print('Rule for ', ?C)
        [all1b: (?Y rdf:type ?D) <- (?X ?P ?Y), (?X rdf:type ?C) ] ]
    
    [max1: (?A rdf:type max(?P, 1)), (?A ?P ?B), (?A ?P ?C)
          -> (?B owl:sameAs ?C) ]
    


- Rule `allID`说明了functor用于将OWL限制的组件收集到单个数据结构中，然后可以触发进一步的规则
- Rule `all2` 表示一个前向规则，它创建了一个新的后向规则，并且还调用了print.
- Rule `max1` 说明了如何使用数字


可以使用以下方法加载和解析规则文件：


    List rules = Rule.rulesFromURL("file:myfile.rules");


或者


    BufferedReader br = / _open reader_ / ;
    List rules = Rule.parseRules( Rule.rulesParserFromReader(br) );


或者


    String ruleSrc = / _list of rules in line_ /
    List rules = Rule.parseRules( rulesSrc );


在前两种情况下（从URL或BufferedReader读取），规则文件由一个简单的处理器预处理，该处理器剥离注释并支持一些额外的宏命令：
<dl><dt><code># ...</code></dt>
<dd>注释.</dd>
<dt><code>// ...</code></dt>
<dd>注释</dd>
<dt><code>@prefix pre: &lt;http://domain/url#&gt;.</code></dt>
<dd>定义了一个前缀<code>pre</code>&nbsp;，可以用在规则文件中.</dd>
<dt><code>@include &lt;urlToRuleFile&gt;.</code></dt>
<dd>包含指定规则,允许规则文件包含RDFS和OWL的预定义规则</dd></dl>
完整实例：


     @prefix pre: <http://jena.hpl.hp.com/prefix#>. 
     @include <RDFS>. 
     [rule1: (?f pre:father ?a) (?u pre:brother ?f) -> (?u pre:uncle ?a)]
     


## 规则推理demo1--喜剧演员

例如，在一个电影知识图谱里，如果一个演员参演的电影的类型是喜剧片，我们可以认为这个演员是喜剧电影

推理规则：


    [ruleComedian: (?p :hasActedIn ?m) (?m :hasGenre ?g) (?g :genreName '喜剧') -> (?p rdf:type :Comedian)] 
    


我们用代码来实现：


     		String prefix = "http://www.test.com/kg/#";
            Graph data = Factory.createGraphMem();
    
            // 定义节点
            Node movie = NodeFactory.createURI(prefix + "movie");
            Node hasActedIn = NodeFactory.createURI(prefix + "hasActedIn");
            Node hasGenre = NodeFactory.createURI(prefix + "hasGenre");
            Node genreName = NodeFactory.createURI(prefix + "genreName");
            Node genre = NodeFactory.createURI(prefix + "genre");
            Node person = NodeFactory.createURI(prefix + "person");
            Node Comedian = NodeFactory.createURI(prefix + "Comedian");
    
            // 添加三元组
            data.add(new Triple(genre, genreName, NodeFactory.createLiteral("喜剧")));
            data.add(new Triple(movie, hasGenre, genre));
            data.add(new Triple(person, hasActedIn, movie));		// 创建推理机
            GenericRuleReasoner reasoner = (GenericRuleReasoner) GenericRuleReasonerFactory.theInstance().create(null);
            PrintUtil.registerPrefix("", prefix);	// 设置规则
            reasoner.setRules(Rule.parseRules(
                    "[ruleComedian: (?p :hasActedIn ?m) (?m :hasGenre ?g) (?g :genreName '喜剧') -> (?p rdf:type :Comedian)] \n"
                            + "-> tableAll()."));
            reasoner.setMode(GenericRuleReasoner.HYBRID); // HYBRID混合推理
    
            InfGraph infgraph = reasoner.bind(data);
            infgraph.setDerivationLogging(true);		// 执行推理
            Iterator<Triple> tripleIterator = infgraph.find(person, null, null);
    
            while (tripleIterator.hasNext()) {
                System.out.println(PrintUtil.print(tripleIterator.next()));
            }
    


输出结果：


    (:person rdf:type :Comedian)
    (:person :hasActedIn :movie)


可以看到，已经给person加上了Comedian。

## 规则推理demo2 -- 关联交易

我们再来看上一篇文章中提到的那个金融图谱：

![金融图谱](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/6/1567758605345.png)

陈华钧老师PPT里，有一个推理任务：

1. 执掌一家公司就一定是这家公司的股东；
2. 某人同时是两家公司的股东，那么这两家公司一定有关联交易；


PPT里是使用Drools来实现的，具体可以参见PPT。我们这里使用jena来实现，可以达到同样的效果。

首先，构造好图谱，为了方便理解，我们用中文变量：


    Model myMod = ModelFactory.createDefaultModel();
            String finance = "http://www.example.org/kse/finance#";
            Resource 孙宏斌 = myMod.createResource(finance + "孙宏斌");
            Resource 融创中国 = myMod.createResource(finance + "融创中国");
            Resource 乐视网 = myMod.createResource(finance + "乐视网");
            Property 执掌 = myMod.createProperty(finance + "执掌");
            Resource 贾跃亭 = myMod.createResource(finance + "贾跃亭");
            Resource 地产公司 = myMod.createResource(finance + "地产公司");
            Resource 公司 = myMod.createResource(finance + "公司");
            Resource 法人实体 = myMod.createResource(finance + "法人实体");
            Resource 人 = myMod.createResource(finance + "人");
            Property 主要收入 = myMod.createProperty(finance + "主要收入");
            Resource 地产事业 = myMod.createResource(finance + "地产事业");
            Resource 王健林 = myMod.createResource(finance + "王健林");
            Resource 万达集团 = myMod.createResource(finance + "万达集团");
            Property 主要资产 = myMod.createProperty(finance + "主要资产");
    
    
            Property 股东 = myMod.createProperty(finance + "股东");
            Property 关联交易 = myMod.createProperty(finance + "关联交易");
            Property 收购 = myMod.createProperty(finance + "收购");
    
            // 加入三元组
            myMod.add(孙宏斌, 执掌, 融创中国);
            myMod.add(贾跃亭, 执掌, 乐视网);
            myMod.add(王健林, 执掌, 万达集团);
            myMod.add(乐视网, RDF.type, 公司);
            myMod.add(万达集团, RDF.type, 公司);
            myMod.add(融创中国, RDF.type, 地产公司);
            myMod.add(地产公司, RDFS.subClassOf, 公司);
            myMod.add(公司, RDFS.subClassOf, 法人实体);
            myMod.add(孙宏斌, RDF.type, 人);
            myMod.add(贾跃亭, RDF.type, 人);
            myMod.add(王健林, RDF.type, 人);
            myMod.add(万达集团,主要资产,地产事业);
            myMod.add(融创中国,主要收入,地产事业);
            myMod.add(孙宏斌, 股东, 乐视网);
            myMod.add(孙宏斌, 收购, 万达集团);
    
            PrintUtil.registerPrefix("", finance);
    
            // 输出当前模型
            StmtIterator i = myMod.listStatements(null,null,(RDFNode)null);
            while (i.hasNext()) {
                System.out.println(" - " + PrintUtil.print(i.nextStatement()));
            }
    


上图所示的图谱，包含如下的三元组：


     - (:公司 rdfs:subClassOf :法人实体)
     - (:万达集团 :主要资产 :地产事业)
     - (:万达集团 rdf:type :公司)
     - (:地产公司 rdfs:subClassOf :公司)
     - (:融创中国 :主要收入 :地产事业)
     - (:融创中国 rdf:type :地产公司)
     - (:孙宏斌 :股东 :乐视网)
     - (:孙宏斌 rdf:type :人)
     - (:孙宏斌 :执掌 :融创中国)
     - (:乐视网 rdf:type :公司)
     - (:贾跃亭 rdf:type :人)
     - (:贾跃亭 :执掌 :乐视网)
     - (:王健林 rdf:type :人)
     - (:王健林 :执掌 :万达集团)


我们来定义推理规则：

1. 执掌一家公司就一定是这家公司的股东；
2. 收购一家公司，就是这家公司的股东
3. 某人同时是两家公司的股东，那么这两家公司一定有关联交易；


用jena规则来表示：


    [ruleHoldShare: (?p :执掌 ?c) -> (?p :股东 ?c)] 
    [[ruleHoldShare2: (?p :收购 ?c) -> (?p :股东 ?c)] 
    [ruleConnTrans: (?p :股东 ?c) (?p :股东 ?c2) -> (?c :关联交易 ?c2)] 


执行推理：


             GenericRuleReasoner reasoner = (GenericRuleReasoner) GenericRuleReasonerFactory.theInstance().create(null);
            reasoner.setRules(Rule.parseRules(
                    "[ruleHoldShare: (?p :执掌 ?c) -> (?p :股东 ?c)] \n"
                            + "[ruleConnTrans: (?p :收购 ?c) -> (?p :股东 ?c)] \n"
                            + "[ruleConnTrans: (?p :股东 ?c) (?p :股东 ?c2) -> (?c :关联交易 ?c2)] \n"
                            + "-> tableAll()."));
            reasoner.setMode(GenericRuleReasoner.HYBRID);
    
            InfGraph infgraph = reasoner.bind(myMod.getGraph());
            infgraph.setDerivationLogging(true);
    
            System.out.println("推理后...\n");
    
            Iterator<Triple> tripleIterator = infgraph.find(null, null, null);
            while (tripleIterator.hasNext()) {
                System.out.println(" - " + PrintUtil.print(tripleIterator.next()));
            }


输出结果：


    
    推理后...
    
     - (:万达集团 :关联交易 :乐视网)
     - (:万达集团 :关联交易 :融创中国)
     - (:万达集团 :关联交易 :万达集团)
     - (:孙宏斌 :股东 :万达集团)
     - (:孙宏斌 :股东 :融创中国)
     - (:融创中国 :关联交易 :万达集团)
     - (:融创中国 :关联交易 :乐视网)
     - (:融创中国 :关联交易 :融创中国)
     - (:乐视网 :关联交易 :万达集团)
     - (:乐视网 :关联交易 :融创中国)
     - (:乐视网 :关联交易 :乐视网)
     - (:贾跃亭 :股东 :乐视网)
     - (:王健林 :股东 :万达集团)
     - (:公司 rdfs:subClassOf :法人实体)
     - (:万达集团 :主要资产 :地产事业)
     - (:万达集团 rdf:type :公司)
     - (:地产公司 rdfs:subClassOf :公司)
     - (:融创中国 :主要收入 :地产事业)
     - (:融创中国 rdf:type :地产公司)
     - (:孙宏斌 :收购 :万达集团)
     - (:孙宏斌 :股东 :乐视网)
     - (:孙宏斌 rdf:type :人)
     - (:孙宏斌 :执掌 :融创中国)
     - (:乐视网 rdf:type :公司)
     - (:贾跃亭 rdf:type :人)
     - (:贾跃亭 :执掌 :乐视网)
     - (:王健林 rdf:type :人)
     - (:王健林 :执掌 :万达集团)


我们看到，推理后孙宏斌是三家公司的股东，三家公司都有关联交易。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


