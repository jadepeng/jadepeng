---
title: 知识图谱推理与实践（1）
tags: ["Jena","知识图谱","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-09-05 15:59
---
文章作者:jqpeng
原文链接: [知识图谱推理与实践（1）](https://www.cnblogs.com/xiaoqi/p/kg-inference-1.html)

由于工作原因，需要在系统里建立图谱推理功能，因此简单学习了浙江大学 陈华钧教授 知识图谱导论课程课件，这里记录下学习笔记。

## 知识图谱推理的主要方法

•  基于描述逻辑的推理（如DL-based）  
 •  基于图结构和统计规则挖掘的推理（如： PRA、 AMIE）  
 •  基于知识图谱表⽰学习的推理（如： TransE）  
 •  基于概率逻辑的⽅法（如： Statistical Relational Learning）

**基于符号逻辑的推理——本体推理**

- 传统的符号逻辑推理中主要与知识图谱有关的推理手段是基于描述逻辑的本体推理。
- 描述逻辑主要被⽤来对事物的本体进⾏建模和推理，⽤来描述和推断概念分类及其概念之间的关系。
- 主要方法：
    - 基于表运算（Tableaux）及改进的⽅法： FaCT++、 Racer、 Pellet	Hermit等
    - 基于Datalog转换的⽅法如KAON、 RDFox等
    - 基于产⽣式规则的算法（如rete）： Jena 、 Sesame、 OWLIM等


![本体推理](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567654565558.png)

**基于图结构和统计规则挖掘的推理**

主要方法：  
 • 基于路径排序学习⽅法(PRA， Path	ranking	Algorithm)	  
 • 基于关联规则挖掘⽅法(AMIE)

![基于图结构和统计规则挖掘的推理](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567654649499.png)

**基于知识图谱表示学习的关系推理**

- 将实体和关系都表示为向量
- 通过向量之间的计算代替图的遍历和搜索来预测三元组的存在，由于向量的表示已经包含了实体原有的语义信息，计算含有⼀定的推理能⼒。
- 可应⽤于链接预测，基于路径的多度查询等


![表示学习推理](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567654752888.png)

**基于概率逻辑的⽅法——Statistical Relational Learning**

概率逻辑学习有时也叫Relational Machine Learning (RML)，关注关系的不确定性和复杂性。  
 通常使用Bayesian networks or Markov networks

![PSL](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567655004326.png)

## 基于符号逻辑的推理

### 本体概念推理

图谱中基于RDF来作为资源描述语言，RDF是**R**esource **D**escription **F**ramework的简称。

![RDF](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567655169157.png)

但是RDF表示关系层次受限，因此有了RDFS,在RDF的基础上，新增了`Class, subClassOf, type, Property, subPropertyOf, Domain, Range` 词汇，可以更好的表述相关关系。

![RDFS](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567655092663.png)

基于RDFS，可以做一些简单的推理

![RDFS推理](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567655311654.png)

OWL在RDFS的基础上，进一步扩展了一些复杂类型、约束：

![OWL](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567655366415.png)

因此，我们也叫OWL为本体语言：

- OWL是知识图谱语言中最规范， 最严谨， 表达能力最强的语言
- 基于RDF语法，使表示出来的文档具有语义理解的结构基础
- 促进了统一词汇表的使用，定义了丰富的语义词汇
- 允许逻辑推理


OWL的描述逻辑系统：

- 一个描述逻辑系统包括四个基本的组成部分
    - 1）最基本的元素： **概念**、**关系**和**个体**（实例），
    - 1. **TBox**术语集 (概念术语的公理集合) - 泛化的知识


        - 描述概念和关系的知识，被称之为公理 (Axiom)
    - 1. **ABox断言集** (个体的断言集合)  --具体个体的信息


        - ABox包含外延知识 (又称断言 (Assertion))，描述论域中  

的特定个体
    - 1. **TBox**和**ABox**上的推理机制
- 不同的描述逻辑系统的表示能力与推理机制由于对这四个组成部分的不同选择而不同


![描述逻辑的语义](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567661842222.png)

描述逻辑与OWL的对应：

![OWL](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567661923597.png)

推理就是通过各种方法获取新的知识或者结论，这些知识和结论满足语义。

OWL本体推理

- 可满足性
    - 本体可满足性： 检查一个本体是否可满足，即检查该本体是否有模型。
    - 概念可满足性，检查某一概念的可满足性，即检查是否有模型，使得对该概念的解释不是空集。


![可满足性](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567662582451.png)

- 分类(classification)，针对Tbox的推理，计算新的概念的包含关系


![分类](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567662678216.png)

![分类](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567662764038.png)

- 实例化（materialization）,即计算属于某个概念或关系的所有实例的集合。


![实例化](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567662840156.png)

例子：

![实例化选股](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567662888484.png)

典型的推理算法： Tableaux，适用于检查某一本体概念的可满足性，以及实例检测，基本思想是通过一系列规则构建Abox，以检测可满足性，或者检测某一实例是否存在于某概念，基本思想类似于一阶逻辑的归结反驳。

![Tableaux](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567663199419.png)

### 基于逻辑编程改写的方法

本体推理的局限:

- (1) 仅支持预定义的本体公理上的推理 (无法针对自定义的词汇支持灵活推理)
- (2) 用户无法定义自己的推理过程


因此，引入规则推理

- (1) 可以根据特定的场景定制规则，以实现用户自定义的推理过程
- (2) Datalog语言可以结合本体推理和规则推理


Datalog的语法：

- 原子（atom）
    - p(t1,t2,...,tn)
    - p是谓词，n是目数，ti是项
    - 例如has\_child(x,y)
- 规则（rule）
    - H:-B1,B2,...,Bm
    - has\_child(X, Y) :−has\_son(X, Y)
- 事实(Fact)
    - F(c1,c2,...cn):-
    - 没有体部且没有变量的规则
    - 例如：has\_child(Alice,Bob):-


Datalog程序是规则的集合：


    has_child(X, Y) : −has_son(X, Y).
    has_child(Alice, Bob) : −


Datalog 推理举例：

![Datalog推理举例](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567663914942.png)

相关工具：

![datalog工具](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567663998133.png)

### 基于产生式规则的方法

产生式系统，一种前向推理系统，可以按照一定机制执行规则从而达到某些目标，与一阶逻辑类似，也有区别，可以应用来做自动规划和专家系统。

产生式系统的组成：

- 事实集合 (Working Memory)
- 产生式/规则集合 (Production Memory, PM)
- 推理引擎


**产生式表示：**

**IF** *conditions* **THEN** *actions*

- conditions是由条件组成的集合，又称为LHS（Left Hand Side）
- actions是由动作组成的序列，又称为RHS（Right Hand Side)


LHS，是条件的集合，各条件是**且**（AND）的关系，当所有条件均被满足，则该规则触发。  
 条件形如(type attr1: spec1 attr2:spec2)条件的形式：

- 原子 (person name:alice)
- 变量（person name:x)
- 表达式 (person age:[n+4]
- 布尔  (person age:{&gt;10})
- 约束的与、或、非


RHS，是执行动作（action）的序列，执行时依次运行。动作的种类有ADD pattern，Remove i，Modify i，可以理解为对WME（Working Memory）的CUD；

产生式举例：


    IF (Student name: x)
    Then ADD (Person name: x)


也可以写作：


    (Student name: x) ⇒ ADD (Person name: x)
    


**推理引擎**

➤ 控制系统的执行：

- 模式匹配，用规则的条件部分匹配事实集中的事实，整个LHS都被满足的规，则被触发，并被加入议程(agenda)
- 解决冲突，按一定的策略从被触发的多条规则中选择一条
- 执行动作，执行被选择出来的规则的RHS，从而对WM进行一定的操作



> 产生式系统=事实集+产生式集合+推理引擎


产生式系统执行流程

![产生式系统执行流程](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567664825865.png)

![匹配举例](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567664993490.png)

模式匹配——RETE算法

- 将产生式的LHS组织成判别网络形式
- 用空间换时间


![RETE算法](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567665661458.png)

相关工具介绍

- Drools
- Jena  提供了处理RDF、 RDFS、 OWL数据的接口，还提供了一个规则引擎



    Model m = ModelFactory.createDefaultModel(); 
    Reasoner reasoner = new
    GenericRuleReasoner(Rule.rulesFromURL("file:rule.txt"));
    InfModel inf = ModelFactory.createInfModel(reasoner, m)


![相关工具](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567666453041.png)

## Inductive Reasoning – 基于图的方法

### PRA

➤ 将连接两个实体的路径作为特征来预测其间可能存在的关系

![Inductive Reasoning](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567666584899.png)

• 通用关系学习框架 (generic relational learning framework)

![PRA](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567666662380.png)

路径排序算法 – Path Ranking Algorithm (PRA)

![PRA2](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567666738103.png)

![enter description here](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667132068.png)

### TransE

知识图谱嵌⼊模型： TransE

TransE(Translating Embeddings for Modeling Multi-relational Data. NIPS 3013)

![TransE](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667166879.png)

![](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667213371.png)

⽬标函数：

![TransE](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667274835.png)

损失函数：

![TransE 损失函数](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667306669.png)

知识图谱嵌⼊模型： 预测问题

- 测试三元组( h, r, t )
- 尾实体预测( h, r, ? )
- 头实体预测( ?, r, t )


![图嵌入预测](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667416512.png)

### PRA vs. TransE

![PRA](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667540090.png)

## 基于Jena实现演绎推理

![实践图谱](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667584143.png)

### 构建model

NO BB, show code：


    Model myMod = ModelFactory.createDefaultModel();
    String finance = “http://www.example.org/kse/finance#”;
    
    // 实体
    Resource shb = myMod.createResource(finance + "孙宏斌");
    Resource rczg = myMod.createResource(finance + "融创中国");
    
    
    // 关系
    
    Property control = myMod.createProperty(finance + "执掌");
    
    // 加入三元组
    myMod.add(shb, control, rczg);
    
    


上图所示的图谱，包含如下的三元组：


    finance :孙宏斌 finance :control finance :融创中国
    finance :贾跃亭 finance :control finance :乐视网
    finance :融创中国 rdf:type finance :地产公司
    finance :地产公司 rdfs:subclassOf finance:公司
    finance:公司 rdfs:subclassOf finance:法人实体
    finance:孙宏斌 rdf:type finance:公司
    finance:孙宏斌 rdf:type finance:人
    finance :人 owl:disjointWith finance:公司


我们可以依次加入，代码略。

### 添加推理机

jena推理使用的是InfModel，可以基于Model构造，实际上在原来的Model之上加了个RDFS推理机


    InfModel inf_rdfs = ModelFactory.createRDFSModel(myMod);
    


• 上下位推理

通过listStatements来获取是否有满足条件的三元组，从而实现判断，subClassOf是RDFS里的vob，因此使用RDFS.subClassOf。


    public static void subClassOf(Model m, Resource s, Resource o) {
    for (StmtIterator i = m.listStatements(s, RDFS.subClassOf, o); i.hasNext(); ) {
    Statement stmt = i.nextStatement();
    System.out.println(" yes! " );
    break;
    }
    }
    
    subClassOf(inf_rdfs, myMod.getResource(finance+"地产公司"),myMod.getResource(finance+”法人实体"));
    


![enter description here](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567667888587.png)

• 针对类别的推理，OWL推理机可以针对个体类别做出完备推理，即补充完整该个体的所有类别；在查询的时候，可以直接打印出所有类别！

首先构建owl推理机：


    Reasoner reasoner = ReasonerRegistry.getOWLReasoner();
    InfModel inf_owl = ModelFactory.createInfModel(reasoner, myMod);
    


然后执行类别推理


    public static void printStatements(Model m, Resource s, Property p, Resource o) {
    for (StmtIterator i = m.listStatements(s,p,o); i.hasNext(); ) {
    Statement stmt = i.nextStatement();
    System.out.println(" - " + PrintUtil.print(stmt));
    }
    }
    printStatements(inf_owl, rczg, RDF.type, null);
    


![enter description here](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567668055162.png)

• 不一致检测, jena的另一个常用推理就是检验data的不一致。


    Model data = FileManager.get().loadModel(fname);
    Reasoner reasoner = ReasonerRegistry.getOWLReasoner();
    InfModel inf_owl = ModelFactory.createInfModel(reasoner, myMod);
    ValidityReport validity = inf_owl.validate();
    if (validity.isValid()) {
    System.out.println(“没有不一致");
    } else {
    System.out.println(“存在不一致，如下： ");
    for (Iterator i = validity.getReports(); i.hasNext(); ) {
    System.out.println(" - " + i.next());
    }
    }


![enter description here](https://gitee.com/jadepeng/pic/raw/master/pic/2019/9/5/1567668144683.png)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


