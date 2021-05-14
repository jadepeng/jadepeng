---
title: Spring Data Mongodb 乐观锁
tags: ["Spring Boot","MongoDB","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-04-16 14:17
---
文章作者:jqpeng
原文链接: [Spring Data Mongodb 乐观锁](https://www.cnblogs.com/xiaoqi/p/spring-data-optimistic-lock.html)

Spring Data 针对mongodb提供了乐观锁实现：


    The @Version annotation provides syntax similar to that of JPA in the context of MongoDB and makes sure updates are only applied to documents with a matching version. Therefore, the actual value of the version property is added to the update query in such a way that the update does not have any effect if another operation altered the document in the meantime. In that case, an OptimisticLockingFailureException is thrown. The following example shows these features:


提供@Version注解，用来标识版本，保存、删除等操作会验证version，不一致会抛出`OptimisticLockingFailureException`

来看一个例子：


    @Document
    class Person {
    
      @Id String id;
      String firstname;
      String lastname;
      @Version Long version;
    }
    
    Person daenerys = template.insert(new Person("Daenerys"));                            (1)
    
    Person tmp = template.findOne(query(where("id").is(daenerys.getId())), Person.class); (2)
    
    daenerys.setLastname("Targaryen");
    template.save(daenerys);                                                              (3)
    
    template.save(tmp); // throws OptimisticLockingFailureException                       (4)


1. 最初插入一个`person` `daenerys`，`version`为`0`。
2. 加载刚插入的数据，`tmp`。`version`还是`0`。
3. 更新`version = 0`的`daenerys`，更新`lastname`，save后`version`变为`1`。
4. 现在来更新，会抛出`OptimisticLockingFailureException`， 提示操作失败。


注意：


| Important | Optimistic Locking requires to set the `WriteConcern` to `ACKNOWLEDGED`. Otherwise `OptimisticLockingFailureException` can be silently swallowed. |
| --- | --- |



| Note | As of Version 2.2 `MongoOperations` also includes the `@Version` property when removing an entity from the database. To remove a `Document` without version check use `MongoOperations#remove(Query,…​)` instead of `MongoOperations#remove(Object)`. |
| --- | --- |



| Note | As of Version 2.2 repositories check for the outcome of acknowledged deletes when removing versioned entities. An `OptimisticLockingFailureException` is raised if a versioned entity cannot be deleted through `CrudRepository.delete(Object)`. In such case, the version was changed or the object was deleted in the meantime. Use `CrudRepository.deleteById(ID)` to bypass optimistic locking functionality and delete objects regardless of their version. |
| --- | --- |


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


