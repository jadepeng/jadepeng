---
title: mongodb海量数据CRUD优化
tags: ["MongoDB","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-05-28 19:47
---
文章作者:jqpeng
原文链接: [mongodb海量数据CRUD优化](https://www.cnblogs.com/xiaoqi/p/java-mongo-crud.html)

## 1. 批量保存优化

避免一条一条查询，采用`bulkWrite`, 基于`ReplaceOneModel`，启用`upsert`:


     public void batchSave(List<?> spoTriples, KgInstance kgInstance) {
            MongoConverter converter = mongoTemplate.getConverter();
            List<ReplaceOneModel<Document>> bulkOperationList = spoTriples.stream()
                    .map(thing -> {
                        org.bson.Document dbDoc = new org.bson.Document();
                        converter.write(thing, dbDoc);
                        ReplaceOneModel<org.bson.Document> replaceOneModel = new ReplaceOneModel(
                                Filters.eq(UNDERSCORE_ID, dbDoc.get(UNDERSCORE_ID)), 
                                dbDoc,
                                new UpdateOptions().upsert(true));
                        return replaceOneModel;
                    })
                    .collect(Collectors.toList());
            mongoTemplate.getCollection(getCollection(kgInstance)).bulkWrite(bulkOperationList);
        }


## 2. 分页优化

经常用于查询的字段，需要确保建立了索引。

对于包含多个键的查询，可以创建符合索引。

### 2.1 避免不必要的count

查询时，走索引，速度并不慢，但是如果返回分页`Page<?>`，需要查询`totalcount`，当单表数据过大时，count会比较耗时，但是设想意向，你真的需要准确的数字吗？

在google、百度等搜索引擎搜索关键词时，只会给你有限的几个结果，因此，我们也不必给出准确的数字，设定一个阈值，比如1万，当我们发现总量大于1万时，返回1万，前端显示大于1万条即可。

原理也很鉴定啊，我们skip掉`MAX_PAGE_COUNT`，看是否还有数据，如果有就说明总量大于`MAX_PAGE_COUNT`，返回`MAX_PAGE_COUNT`即可，否则，计算真正的count。


    
    
    int MAX_PAGE_COUNT = 10000;
    
    
    /**
         * 当总数大于阈值时，不再计算总数
         *
         * @param mongoTemplate
         * @param query
         * @param collectionName
         * @return
         */
        private long count(MongoTemplate mongoTemplate, Query query, String collectionName) {
            query = query.with(PageRequest.of(MAX_PAGE_COUNT, 1));
            if (mongoTemplate.find(query, Thing.class, collectionName).size() > 0) {
                return MAX_PAGE_COUNT;
            }
            return mongoTemplate.count(query, collectionName);
        }


前端显示：

![大于10000](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-28/1559041614698.png)

### 2.2 避免过多的skip

分页不过避免需要先跳过一些数据，这个过程是需要消耗时间的，可以通过一个小技巧避免跳过。

比如，显示列表时，排序为按最后修改时间倒序，每页显示100条，现在要显示第100页。  
 按照正常的做法，需要跳过`99*100`条数据，非常大的代价。换一个角度思考，因为数据是有序的，因此第100页的数据的最后修改时间是小于第99页最小的修改时间，查询时加上这个条件，就可以直接取符合条件的前100条即可。

## 3. 全量导出优化

### 3.1 去掉不需要的字段

查询时，指定真正有用的字段，这样可以有效减少数据传输量，加快查询效率。  
 例如：


     	    Query query = new Query();
            query.fields().include("_id").include("name").include("hot").include("alias");


### 3.2 避免使用findAll或者分页查询，改用stream

全量导出有两个误区，一是直接`findAll`,当数据量过大时，很容易导致服务器`OutofMermory`，就算没有OOM，也会对服务器造成极大的负载，影响兄弟服务。另外，FindAll一次性加载数据到内存，整个速度也会比较慢，需要等待所有数据进入内存后才能开始处理。

另外一个误区是，分页查询，依次处理。分页查询可以有效减少服务器负担，不失为一种可行的方法。但是就和上面分页说的那样，分页到后面的时候，需要skip掉前面的数据，存在无用功。稍微好一点的做法就是按照之前说的，将skip转换为condtion，这种方式效率OK，但不推荐，存在代码冗余。


                Page<Thing> dataList = entityDao.findAllByPage(kgDataStoreService.getKgCollectionByKgInstance(kg), page);
                Map<String, Individual> thingId2Resource = new ConcurrentHashMap<>();
    
                appendThingsToModel(model, concept2OntClass, hot, alias, dataList, thingId2Resource);
    
                while (dataList.hasNext()) {
                    page = PageRequest.of(page.getPageNumber() + 1, page.getPageSize());
                    dataList = entityDao.findAllByPage(kgDataStoreService.getKgCollectionByKgInstance(kg), page);
                    appendThingsToModel(model, concept2OntClass, hot, alias, dataList, thingId2Resource);
                }


更推荐的做法是，采用mongoTemplate的steam方法,返回`CloseableIterator`迭代器，读一条数据处理一条数据，实现高效处理：


    @Overridepublic <T> CloseableIterator<T> stream(final Query query, final Class<T> entityType, final String collectionName) {	return doStream(query, entityType, collectionName, entityType);}


改用方法后，代码可以更简化高效：


      CloseableIterator<Thing> dataList = kgDataStoreService.getSimpleInfoIterator(kg);
    
                // 实体导入
                // Page<Thing> dataList = entityDao.findAllByPage(kgDataStoreService.getKgCollectionByKgInstance(kg), page);
                Map<String, Individual> thingId2Resource = new ConcurrentHashMap<>();
    
                appendThingsToModel(model, concept2OntClass, hot, alias, dataList, thingId2Resource);


待续。。。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


