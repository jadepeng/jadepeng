---
title: 知识图谱推理与实践(3) -- jena自定义builtin
tags: ["Jena","知识图谱","推理","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-09-12 09:29
---
文章作者:jqpeng
原文链接: [知识图谱推理与实践(3) -- jena自定义builtin](https://www.cnblogs.com/xiaoqi/p/kg-inference-3.html)

在[第2篇](https://www.cnblogs.com/xiaoqi/p/kg-inference-2.html)里，介绍了`jena`的`The general purpose rule engine`（通用规则引擎）及其使用，本篇继续探究，如何自定义builtin。

## builtin介绍

先回顾builtin为何物，官方叫`Builtin primitives`,可以理解为内置函数、内置指令，可以返回true或者false用来检验rule是否匹配，官方包含如下的`primitives`




| Builtin | Operations |
| --- | --- |
| isLiteral(?x) notLiteral(?x)             <br>isFunctor(?x) notFunctor(?x)             <br>isBNode(?x) notBNode(?x) | Test whether the single argument is or is not a literal, a functor-valued
          literal or a blank-node, respectively. |
| bound(?x...) unbound(?x..) | Test if all of the arguments are bound (not bound) variables |
| equal(?x,?y) notEqual(?x,?y) | Test if x=y (or x != y). The equality test is semantic equality so that,
          for example, the xsd:int 1 and the xsd:decimal 1 would test equal. |
| lessThan(?x, ?y), greaterThan(?x, ?y)             <br>le(?x, ?y), ge(?x, ?y) | Test if x is &lt;, &gt;, &lt;= or &gt;= y. Only passes if both x and y
          are numbers or time instants (can be integer or floating point or XSDDateTime). |
| sum(?a, ?b, ?c)             <br>addOne(?a, ?c)             <br>difference(?a, ?b, ?c)             <br>min(?a, ?b, ?c)             <br>max(?a, ?b, ?c)             <br>product(?a, ?b, ?c)             <br>quotient(?a, ?b, ?c) | Sets c to be (a+b), (a+1) (a-b), min(a,b), max(a,b), (a
          *b), (a/b). Note that these do not run backwards, if in
             `sum` a and c are bound and b is unbound then the test will
            fail rather than bind b to (c-a). This could be fixed.*<br> |
| strConcat(?a1, .. ?an, ?t)             <br>uriConcat(?a1, .. ?an, ?t) | Concatenates the lexical form of all the arguments except the last, then
          binds the last argument to a plain literal (strConcat) or a URI node
          (uriConcat) with that lexical form. In both cases if an argument node
          is a URI node the URI will be used as the lexical form. |
| regex(?t, ?p)             <br>regex(?t, ?p, ?m1, .. ?mn) | Matches the lexical form of a literal (?t) against a regular expression
          pattern given by another literal (?p). If the match succeeds, and if
          there are any additional arguments then it will bind the first n capture
          groups to the arguments ?m1 to ?mn. The regular expression pattern syntax
          is that provided by java.util.regex. Note that the capture groups are
          numbered from 1 and the first capture group will be bound to ?m1, we
          ignore the implicit capture group 0 which corresponds to the entire matched
          string. So for example
          <br><br>    regexp('foo bar', '(.) (.<br>                )', ?m1, ?m2)<br>              <br><br>*will bind
             `m1` to
             `"foo"` and
             `m2` to
             `"bar"`.*<br> |
| now(?x) | Binds ?x to an xsd:dateTime value corresponding to the current time. |
| makeTemp(?x) | Binds ?x to a newly created blank node. |
| makeInstance(?x, ?p, ?v)
          <br>makeInstance(?x, ?p, ?t, ?v) | Binds ?v to be a blank node which is asserted as the value of the ?p property
          on resource ?x and optionally has type ?t. Multiple calls with the same
          arguments will return the same blank node each time - thus allowing this
          call to be used in backward rules. |
| makeSkolem(?x, ?v1, ... ?vn) | Binds ?x to be a blank node. The blank node is generated based on the values
          of the remain ?vi arguments, so the same combination of arguments will
          generate the same bNode. |
| noValue(?x, ?p)
          <br>noValue(?x ?p ?v) | True if there is no known triple (x, p, ) or (x, p, v) in the model or
          the explicit forward deductions so far. |
| remove(n, ...)
          <br>drop(n, ...) | Remove the statement (triple) which caused the n'th body term of this (forward-only)
          rule to match. Remove will propagate the change to other consequent rules
          including the firing rule (which must thus be guarded by some other clauses).
          In particular, if the removed statement (triple) appears in the body
          of a rule that has already fired, the consequences of such rule are retracted
          from the deducted model. Drop will silently remove the triple(s) from
          the graph but not fire any rules as a consequence. These are clearly
          non-monotonic operations and, in particular, the behaviour of a rule
          set in which different rules both drop and create the same triple(s)
          is undefined. |
| isDType(?l, ?t) notDType(?l, ?t) | Tests if literal ?l is (or is not) an instance of the datatype defined
          by resource ?t. |
| print(?x, ...) | Print (to standard out) a representation of each argument. This is useful
          for debugging rather than serious IO work. |
| listContains(?l, ?x)
           <br>listNotContains(?l, ?x) | Passes if ?l is a list which contains (does not contain) the element ?x,
          both arguments must be ground, can not be used as a generator. |
| listEntry(?list, ?index, ?val) | Binds ?val to the ?index'th entry in the RDF list ?list. If there is no
          such entry the variable will be unbound and the call will fail. Only
          usable in rule bodies. |
| listLength(?l, ?len) | Binds ?len to the length of the list ?l. |
| listEqual(?la, ?lb)
           <br>listNotEqual(?la, ?lb) | listEqual tests if the two arguments are both lists and contain the same
          elements. The equality test is semantic equality on literals (sameValueAs)
          but will not take into account owl:sameAs aliases. listNotEqual is the
          negation of this (passes if listEqual fails). |
| listMapAsObject(?s, ?p ?l)
           <br>listMapAsSubject(?l, ?p, ?o) | These can only be used as actions in the head of a rule. They deduce a
          set of triples derived from the list argument ?l : listMapAsObject asserts
          triples (?s ?p ?x) for each ?x in the list ?l, listMapAsSubject asserts
          triples (?x ?p ?o). |
| table(?p) tableAll() | Declare that all goals involving property ?p (or all goals) should be tabled
          by the backward engine. |
| hide(p) | Declares that statements involving the predicate p should be hidden. Queries
          to the model will not report such statements. This is useful to enable
          non-monotonic forward rules to define flag predicates which are only
          used for inference control and do not "pollute" the inference results. |

  

## builtin 自定义

自定义很简单，实现`Builtin`接口, 然后使用`BuiltinRegistry.theRegistry.register`注册即可。

Builtin接口定义如下：


    public interface Builtin {
    
        /**
         * Return a convenient name for this builtin, normally this will be the name of the 
         * functor that will be used to invoke it and will often be the final component of the
         * URI.
         */
        public String getName();
        
        /**
         * Return the full URI which identifies this built in.
         */
        public String getURI();
        
        /**
         * Return the expected number of arguments for this functor or 0 if the number is flexible.
         */
        public int getArgLength();
        
        /**
         * This method is invoked when the builtin is called in a rule body.
         * @param args the array of argument values for the builtin, this is an array 
         * of Nodes, some of which may be Node_RuleVariables.
         * @param length the length of the argument list, may be less than the length of the args array
         * for some rule engines
         * @param context an execution context giving access to other relevant data
         * @return return true if the buildin predicate is deemed to have succeeded in
         * the current environment
         */
        public boolean bodyCall(Node[] args, int length, RuleContext context);
        
        /**
         * This method is invoked when the builtin is called in a rule head.
         * Such a use is only valid in a forward rule.
         * @param args the array of argument values for the builtin, this is an array 
         * of Nodes.
         * @param length the length of the argument list, may be less than the length of the args array
         * for some rule engines
         * @param context an execution context giving access to other relevant data
         */
        public void headAction(Node[] args, int length, RuleContext context);
        
        /**
         * Returns false if this builtin has side effects when run in a body clause,
         * other than the binding of environment variables.
         */
        public boolean isSafe();
        
        /**
         * Returns false if this builtin is non-monotonic. This includes non-monotonic checks like noValue
         * and non-monotonic actions like remove/drop. A non-monotonic call in a head is assumed to 
         * be an action and makes the overall rule and ruleset non-monotonic. 
         * Most JenaRules are monotonic deductive closure rules in which this should be false.
         */
        public boolean isMonotonic();
    }
    


一般我们不用直接实现该接口，可以继承默认的实现`BaseBuiltin`, 一般只需要Override 下`getName`提供指令名称，实现`bodyCall`,提供函数调用即可。


        @Override
        public String getName() {
            return "semsim";
        }
    


比如，我们来自定义一个指令，用来计算两两语义相似度：


    public class SemanticSimilarityBuiltin extends BaseBuiltin {
        /**
         * Return a convenient name for this builtin, normally this will be the name of the
         * functor that will be used to invoke it and will often be the final component of the
         * URI.
         */
        @Override
        public String getName() {
            return "semsim";
        }
    
        @Override
        public int getArgLength() {
            return 3;
        }
    
    
        /**
         * This method is invoked when the builtin is called in a rule body.
         *
         * @param args    the array of argument values for the builtin, this is an array
         *                of Nodes, some of which may be Node_RuleVariables.
         * @param context an execution context giving access to other relevant data
         * @return return true if the buildin predicate is deemed to have succeeded in
         * the current environment
         */
        @Override
        public boolean bodyCall(Node[] args, int length, RuleContext context) {
            checkArgs(length, context);
            Node n1 = getArg(0, args, context);
            Node n2 = getArg(1, args, context);
            Node score = getArg(2,args,context);
    
            if(!score.isLiteral()  || score.getLiteral().getValue()==null){
             return false;
            }
            String value;
            Double hold = Double.parseDouble(score.getLiteralValue().toString());
    
            //  n.isLiteral() && n.getLiteralValue() instanceof Number
    
            if (n1.isLiteral() && n2.isLiteral()) {
                String v1 = n1.getLiteralValue().toString();
                String v2 = n2.getLiteralValue().toString();
    
                // 调用服务计算相似度
                String requestUrl = "http://API-URL:5101/similarity/cosine?s1="+v1+"&s2="+v2;
                String result = HttpClientUtil.doGet(requestUrl);
                JSONObject json = JSON.parseObject(result);
                if(json.getDouble("similarity") >= hold){
                    return true;
                }
    
                return true;
            }
            return false;
        }
    }


- 这里有个getArgLength和checkArgs(length, context)，可以用来限制参数长度，检验必须符合该长度。
- 可以通过getArg(idx, args, context)来获取待计算的参数
- 上面的计算相似度，主要是调用外度的服务来计算两两的语义向量的cosine得分，如果满足阈值，我们就认为规则匹配


## 测试

我们来测试上面的定义的计算语义相似度的指令`semsim`，还是用第2篇里的例子：

我们新增加两个属性`主要业务`和`竞争对手`，我们定义，如果两个公司的主要业务语义上相似，我们就认为两家公司是竞争对手。


            Property 主要业务 = myMod.createProperty(finance + "主要业务");
            Property 竞争对手 = myMod.createProperty(finance + "竞争对手");
    
            // 加入三元组
          
            myMod.add(万达集团, 主要业务, "房地产，文娱");
            myMod.add(融创中国, 主要业务, "房地产");
          


然后定义规则：


    [ruleCompetitor: (?c1 :主要业务 ?b1) (?c2 :主要业务 ?b2) notEqual(?c1,?c2) semsim(?b1,?b2,0.6)  -> (?c1 :竞争对手 ?c2)] 


规则意思是，公司C1 主要业务是 b1,c2 主要业务是b2,并且c1和c2不是同一家公司，如果b1，b2的相似度大于0.6，那么C1和c2是竞争对手。

完整测试代码：


           // 注册自定义builtin
            BuiltinRegistry.theRegistry.register(new SemanticSimilarityBuiltin());
    
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
    
            Property 主要业务 = myMod.createProperty(finance + "主要业务");
            Property 竞争对手 = myMod.createProperty(finance + "竞争对手");
    
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
            myMod.add(万达集团, 主要资产, 地产事业);
            myMod.add(万达集团, 主要业务, "房地产，文娱");
            myMod.add(融创中国, 主要收入, 地产事业);
            myMod.add(融创中国, 主要业务, "房地产");
            myMod.add(孙宏斌, 股东, 乐视网);
            myMod.add(孙宏斌, 收购, 万达集团);
    
            PrintUtil.registerPrefix("", finance);
    
            // 输出当前模型
            StmtIterator i = myMod.listStatements(null, null, (RDFNode) null);
            while (i.hasNext()) {
                System.out.println(" - " + PrintUtil.print(i.nextStatement()));
            }
    
    
            GenericRuleReasoner reasoner = (GenericRuleReasoner) GenericRuleReasonerFactory.theInstance().create(null);
            reasoner.setRules(Rule.parseRules(
                "[ruleHoldShare: (?p :执掌 ?c) -> (?p :股东 ?c)] \n"
                    + "[ruleConnTrans: (?p :收购 ?c) -> (?p :股东 ?c)] \n"
                    + "[ruleConnTrans: (?p :股东 ?c) (?p :股东 ?c2) -> (?c :关联交易 ?c2)] \n"
                    + "[ruleCompetitor:: (?c1 :主要业务 ?b1) (?c2 :主要业务 ?b2) notEqual(?c1,?c2) semsim(?b1,?b2,0.6)  -> (?c1 :竞争对手 ?c2)] \n"
                    + "-> tableAll()."));
            reasoner.setMode(GenericRuleReasoner.HYBRID);
    
            InfGraph infgraph = reasoner.bind(myMod.getGraph());
            infgraph.setDerivationLogging(true);
    
            System.out.println("推理后...\n");
    
            Iterator<Triple> tripleIterator = infgraph.find(null, null, null);
            while (tripleIterator.hasNext()) {
                System.out.println(" - " + PrintUtil.print(tripleIterator.next()));
            }


运行结果：


     - (:万达集团 :关联交易 :乐视网)
     - (:万达集团 :关联交易 :融创中国)
     - (:万达集团 :竞争对手 :融创中国)
     - (:万达集团 :关联交易 :万达集团)
     - (:孙宏斌 :股东 :万达集团)
     - (:孙宏斌 :股东 :融创中国)
     - (:融创中国 :关联交易 :万达集团)
     - (:融创中国 :竞争对手 :万达集团)
     - (:融创中国 :关联交易 :乐视网)
     - (:融创中国 :关联交易 :融创中国)
     - (:乐视网 :关联交易 :万达集团)
     - (:乐视网 :关联交易 :融创中国)
     - (:乐视网 :关联交易 :乐视网)
     - (:贾跃亭 :股东 :乐视网)
     - (:王健林 :股东 :万达集团)
     - (:公司 rdfs:subClassOf :法人实体)
     - (:万达集团 :主要业务 '房地产，文娱')
     - (:万达集团 :主要资产 :地产事业)
     - (:万达集团 rdf:type :公司)
     - (:地产公司 rdfs:subClassOf :公司)
     - (:融创中国 :主要业务 '房地产')
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


可以根据需要，扩展更多的builtin，比如运行js，比如http请求。。。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


