---
title: 使用Spring访问Mongodb的方法大全——Spring Data MongoDB查询指南
tags: ["Spring","MongoDB","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-12-26 16:28
---
文章作者:jqpeng
原文链接: [使用Spring访问Mongodb的方法大全——Spring Data MongoDB查询指南](https://www.cnblogs.com/xiaoqi/p/queries-in-spring-data-mongodb.html)

# 1.概述

Spring Data MongoDB 是Spring框架访问mongodb的神器，借助它可以非常方便的读写mongo库。本文介绍使用Spring Data MongoDB来访问mongodb数据库的几种方法：

- 使用Query和Criteria类
- JPA自动生成的查询方法
- 使用@Query 注解基于JSON查询


在开始前，首先需要引入maven依赖

## 1.1 添加Maven的依赖

如果您想使用Spring Data MongoDB，则需要将以下条目添加到您的pom.xml文件中：


    <dependency><groupId>org.springframework.data</groupId><artifactId>spring-data-mongodb</artifactId><version>1.9.6.RELEASE</version>
    </dependency>


版本根据需要选择。

# 2.文档查询

使用Spring Data来查询MongoDB的最常用方法之一是使用Query和Criteria类 ， 它们非常接近本地操作符。

## 2.1 is查询

在以下示例中 - 我们正在寻找名为Eric的用户。

我们来看看我们的数据库：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45},{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 55}
    }]


让我们看看查询代码：


    Query query = new Query();
    query.addCriteria(Criteria.where("name").is("Eric"));
    List<User> users = mongoTemplate.find(query, User.class);


如预期的那样，这个逻辑返回：


    {"_id" : ObjectId("55c0e5e5511f0a164a581907"),"_class" : "org.baeldung.model.User","name" : "Eric","age" : 45
    }


## 2.2 正则查询

正则表达式是一个更灵活和强大的查询类型。这使用了一个使用MongoDB $ regex的标准，该标准返回适用于这个字段的这个正则表达式的所有记录。

它的作用类似于startingWith，endingWith操作 - 让我们来看一个例子。

寻找名称以A开头的所有用户，这是数据库的状态：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45},{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33},{	"_id" : ObjectId("55c0e5e5511f0a164a581909"),	"_class" : "org.baeldung.model.User",	"name" : "Alice",	"age" : 35}
    ]


我们来创建查询：


    Query query = new Query();
    query.addCriteria(Criteria.where("name").regex("^A"));
    List<User> users = mongoTemplate.find(query,User.class);


这运行并返回2条记录：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33},{	"_id" : ObjectId("55c0e5e5511f0a164a581909"),	"_class" : "org.baeldung.model.User",	"name" : "Alice",	"age" : 35}
    ]


下面是另一个简单的例子，这次查找名称以c结尾的所有用户：


    Query query = new Query();
    query.addCriteria(Criteria.where("name").regex("c$"));
    List<User> users = mongoTemplate.find(query, User.class);


所以结果是：


    {"_id" : ObjectId("55c0e5e5511f0a164a581907"),"_class" : "org.baeldung.model.User","name" : "Eric","age" : 45
    }


## 2.3 LT和GT

$ lt（小于）运算符和$ gt（大于）。

让我们快速看一个例子 - 我们正在寻找年龄在20岁到50岁之间的所有用户。

数据库是：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45},{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 55}
    }


构造查询：


    Query query = new Query();
    query.addCriteria(Criteria.where("age").lt(50).gt(20));
    List<User> users = mongoTemplate.find(query,User.class);


结果 - 年龄大于20且小于50的所有用户：


    {"_id" : ObjectId("55c0e5e5511f0a164a581907"),"_class" : "org.baeldung.model.User","name" : "Eric","age" : 45
    }


## 2.4 结果排序

Sort用于指定结果的排序顺序。

首先 - 这里是现有的数据：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45},{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33},{	"_id" : ObjectId("55c0e5e5511f0a164a581909"),	"_class" : "org.baeldung.model.User",	"name" : "Alice",	"age" : 35}
    ]


执行排序后：


    Query query = new Query();
    query.with(new Sort(Sort.Direction.ASC, "age"));
    List<User> users = mongoTemplate.find(query,User.class);


这是查询的结果 - 很好地按年龄排序：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33},{	"_id" : ObjectId("55c0e5e5511f0a164a581909"),	"_class" : "org.baeldung.model.User",	"name" : "Alice",	"age" : 35},{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45}
    ]


## 2.5 分页

我们来看一个使用分页的简单例子。

这是数据库的状态：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45},{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33},{	"_id" : ObjectId("55c0e5e5511f0a164a581909"),	"_class" : "org.baeldung.model.User",	"name" : "Alice",	"age" : 35}
    ]


现在，查询逻辑，只需要一个大小为2的页面：


    final Pageable pageableRequest = new PageRequest(0, 2);
    Query query = new Query();
    query.with(pageableRequest);


结果 ：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581907"),	"_class" : "org.baeldung.model.User",	"name" : "Eric",	"age" : 45},{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33}
    ]


为了探索这个API的全部细节，这里是Query和Criteria类的文档。

## 3.生成的查询方法（Generated Query Methods）

生成查询方法是JPA的一个特性，在Spring Data Mongodb里也可以使用。

要做到2里功能，只需要在接口上声明方法即可，


    public interface UserRepository 
      extends MongoRepository<User, String>, QueryDslPredicateExecutor<User> {...
    }


## 3.1 FindByX

我们将通过探索findBy类型的查询来简单地开始 - 在这种情况下，通过名称查找：


    List<User> findByName(String name);


与上一节相同 2.1 - 查询将具有相同的结果，查找具有给定名称的所有用户：


    List<User> users = userRepository.findByName("Eric");


## 3.2  StartingWith and endingWith.

下面是操作过程的一个简单例子：


    List<User> findByNameStartingWith(String regexp);
    
    List<User> findByNameEndingWith(String regexp);


实际使用这个例子当然会非常简单：


    List<User> users = userRepository.findByNameStartingWith("A");
    List<User> users = userRepository.findByNameEndingWith("c");


结果是完全一样的。

## 3.3 Between

类似于2.3，这将返回年龄在ageGT和ageLT之间的所有用户：


    List<User> findByAgeBetween(int ageGT, int ageLT);
    List<User> users = userRepository.findByAgeBetween(20, 50);


## 3.4 Like和OrderBy

让我们来看看这个更高级的示例 - 为生成的查询组合两种类型的修饰符。

我们将要查找名称中包含字母A的所有用户，我们也将按年龄顺序排列结果：


    List<User> users = userRepository.findByNameLikeOrderByAgeAsc("A");


结果：


    [{	"_id" : ObjectId("55c0e5e5511f0a164a581908"),	"_class" : "org.baeldung.model.User",	"name" : "Antony",	"age" : 33},{	"_id" : ObjectId("55c0e5e5511f0a164a581909"),	"_class" : "org.baeldung.model.User",	"name" : "Alice",	"age" : 35}
    ]


# 4. JSON查询方法

如果我们无法用方法名称或条件来表示查询，那么我们可以做更低层次的事情 - 使用@Query注解。

通过这个注解，我们可以指定一个原始查询 - 作为一个Mongo JSON查询字符串。

## 4.1 FindBy

让我们先从简单的，看看我们是如何将是一个通过查找类型的方法第一：


    @Query("{ 'name' : ?0 }")
    List<User> findUsersByName(String name);


这个方法应该按名称返回用户 - 占位符?0引用方法的第一个参数。

## 4.2 $regex

让我们来看一个正则表达式驱动的查询 - 这当然会产生与2.2和3.2相同的结果：


    @Query("{ 'name' : { $regex: ?0 } }")
    List<User> findUsersByRegexpName(String regexp);


用法也完全一样：


    List<User> users = userRepository.findUsersByRegexpName("^A");
    List<User> users = userRepository.findUsersByRegexpName("c$");


## 4.3. $ lt和$ gt

现在我们来实现lt和gt查询：


    @Query("{ 'age' : { $gt: ?0, $lt: ?1 } }")
    List<User> findUsersByAgeBetween(int ageGT, int ageLT);


# 5. 结论

在本文中，我们探讨了使用Spring Data MongoDB进行查询的常用方法。

本文示例可以从 [spring-data-mongodb](https://github.com/eugenp/tutorials/tree/master/spring-data-mongodb)这里下载。

本文参考[A Guide to Queries in Spring Data MongoDB](http://www.baeldung.com/queries-in-spring-data-mongodb)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


