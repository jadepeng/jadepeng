---
title:  hugegraph 索引与查询优化分析
tags: ["hugegraph"]
categories: ["博客"]
date: 2021-07-12 21:00
---


## 为什么要有索引

`gremlin` 其实是一个逐级过滤的运行机制，比如下面的一个简单的gremlin查询语句：

```
g.V().hasLabel("label").has("prop","value")
```

运行原理就是：

- 找出所有的顶点V
- 然后过滤出label为label的数据
- 然后过滤出prop=value的数据

当数据量很大时，这个代价非常大，因此需要做查询优化。

`hugegraph` 的优化方案是，`HugeGraphStepStrategy` 中将has条件提取出来，然后走索引优化，减少读取的数据量。

`TraversalUtil.extractHasContainer`:

```java
 public static void extractHasContainer(HugeGraphStep<?, ?> newStep,
                                           Traversal.Admin<?, ?> traversal) {
        Step<?, ?> step = newStep;
        do {
            step = step.getNextStep();
            if (step instanceof HasStep) {
                HasContainerHolder holder = (HasContainerHolder) step;
                for (HasContainer has : holder.getHasContainers()) {
                    if (!GraphStep.processHasContainerIds(newStep, has)) {
                        newStep.addHasContainer(has);
                    }
                }
                TraversalHelper.copyLabels(step, step.getPreviousStep(), false);
                traversal.removeStep(step);
            }
        } while (step instanceof HasStep || step instanceof NoOpBarrierStep);
    }
```


## hugegraph索引介绍

`hugegraph` 通过`IndexLabel` 来定义索引类型，描述索引的约束信息。

*   indexType: 建立的索引类型，目前支持五种，即 Secondary、Range、Search、Shard 和 Unique。
    *   Secondary 支持精确匹配的二级索引，允许建立联合索引，联合索引支持索引前缀搜索
        *   单个属性，支持相等查询，比如：person顶点的city属性的二级索引，可以用`g.V().has("city", "北京")`查询"city属性值是北京"的全部顶点
        *   联合索引，支持前缀查询和相等查询，比如：person顶点的city和street属性的联合索引，可以用`g.V().has ("city", "北京").has('street', '中关村街道')`查询"city属性值是北京且street属性值是中关村"的全部顶点，或者`g.V() .has("city", "北京")`查询"city属性值是北京"的全部顶点

            > 
            > 
            > secondary index的查询都是基于"是"或者"相等"的查询条件，不支持"部分匹配"
            > 
            > 

    *   Range 支持数值类型的范围查询
        *   必须是单个数字或者日期属性，比如：person顶点的age属性的范围索引，可以用`g.V().has("age", P.gt(18))`查询"age属性值大于18"的顶点。除了`P.gt()`以外，还支持`P.gte()`, `P.lte()`, `P.lt()`, `P.eq()`, `P.between()`, `P.inside()`和`P.outside()`等
    *   Search 支持全文检索的索引
        *   必须是单个文本属性，比如：person顶点的address属性的全文索引，可以用`g.V().has("address", Text .contains('大厦')`查询"address属性中包含大厦"的全部顶点

            > 
            > 
            > search index的查询是基于"是"或者"包含"的查询条件
            > 
            > 

    *   Shard 支持前缀匹配 + 数字范围查询的索引
        *   N个属性的分片索引，支持前缀相等情况下的范围查询，比如：person顶点的city和age属性的分片索引，可以用`g.V().has ("city", "北京").has("age", P.between(18, 30))`查询"city属性是北京且年龄大于等于18小于30"的全部顶点
        *   shard index N个属性全是文本属性时，等价于secondary index
        *   shard index只有单个数字或者日期属性时，等价于range index

            > 
            > 
            > shard index可以有任意数字或者日期属性，但是查询时最多只能提供一个范围查找条件，且该范围查找条件的属性的前缀属性都是相等查询条件
            > 
            > 

    *   Unique 支持属性值唯一性约束，即可以限定属性的值不重复，允许联合索引，但不支持查询
        *   单个或者多个属性的唯一性索引，不可用来查询，只可对属性的值进行限定，当出现重复值时将报错


> 摘录自 https://hugegraph.github.io/hugegraph-doc/clients/hugegraph-client.html

`Secondary` 和 `Range`是最常用的索引。


## 索引存储原理


我们通过源代码来分析索引存储过程。 核心代码在`GraphIndexTransaction.updateIndex`函数里：

```java
/**
     * Update index(user properties) of vertex or edge
     * @param ilId      the id of index label
     * @param element   the properties owner
     * @param removed   remove or add index
     */
    protected void updateIndex(Id ilId, HugeElement element, boolean removed) {
        SchemaTransaction schema = this.params().schemaTransaction();
        IndexLabel indexLabel = schema.getIndexLabel(ilId);
        E.checkArgument(indexLabel != null,
                        "Not exist index label with id '%s'", ilId);

        // Collect property values of index fields
        List<Object> allPropValues = new ArrayList<>();
        int fieldsNum = indexLabel.indexFields().size();
        int firstNullField = fieldsNum;
        for (Id fieldId : indexLabel.indexFields()) {
            HugeProperty<Object> property = element.getProperty(fieldId);
            if (property == null) {
                E.checkState(hasNullableProp(element, fieldId),
                             "Non-null property '%s' is null for '%s'",
                             this.graph().propertyKey(fieldId) , element);
                if (firstNullField == fieldsNum) {
                    firstNullField = allPropValues.size();
                }
                allPropValues.add(INDEX_SYM_NULL);
            } else {
                E.checkArgument(!INDEX_SYM_NULL.equals(property.value()),
                                "Illegal value of index property: '%s'",
                                INDEX_SYM_NULL);
                allPropValues.add(property.value());
            }
        }

        if (firstNullField == 0 && !indexLabel.indexType().isUnique()) {
            // The property value of first index field is null
            return;
        }
        // Not build index for record with nullable field (except unique index)
        List<Object> propValues = allPropValues.subList(0, firstNullField);

        // Expired time
        long expiredTime = element.expiredTime();

        // Update index for each index type
        switch (indexLabel.indexType()) {
            case RANGE_INT:
            case RANGE_FLOAT:
            case RANGE_LONG:
            case RANGE_DOUBLE:
                E.checkState(propValues.size() == 1,
                             "Expect only one property in range index");
                Object value = NumericUtil.convertToNumber(propValues.get(0));
                this.updateIndex(indexLabel, value, element.id(),
                                 expiredTime, removed);
                break;
            case SEARCH:
                E.checkState(propValues.size() == 1,
                             "Expect only one property in search index");
                value = propValues.get(0);
                Set<String> words = this.segmentWords(value.toString());
                for (String word : words) {
                    this.updateIndex(indexLabel, word, element.id(),
                                     expiredTime, removed);
                }
                break;
            case SECONDARY:
                // Secondary index maybe include multi prefix index
                for (int i = 0, n = propValues.size(); i < n; i++) {
                    List<Object> prefixValues = propValues.subList(0, i + 1);
                    // prefixValues is list or set , should create index for
                    // each item
                    if(prefixValues.get(0) instanceof Collection) {
                        for (Object propValue :
                                (Collection<Object>) prefixValues.get(0)) {
                            value = escapeIndexValueIfNeeded(propValue.toString());
                            this.updateIndex(indexLabel, value, element.id(),
                                             expiredTime, removed);
                        }
                    }else {
                        value = ConditionQuery.concatValues(prefixValues);
                        value = escapeIndexValueIfNeeded((String) value);
                        this.updateIndex(indexLabel, value, element.id(),
                                         expiredTime, removed);
                    }
                }
                break;
            case SHARD:
                value = ConditionQuery.concatValues(propValues);
                value = escapeIndexValueIfNeeded((String) value);
                this.updateIndex(indexLabel, value, element.id(),
                                 expiredTime, removed);
                break;
            case UNIQUE:
                value = ConditionQuery.concatValues(allPropValues);
                assert !value.equals("");
                Id id = element.id();
                // TODO: add lock for updating unique index
                if (!removed && this.existUniqueValue(indexLabel, value, id)) {
                    throw new IllegalArgumentException(String.format(
                              "Unique constraint %s conflict is found for %s",
                              indexLabel, element));
                }
                this.updateIndex(indexLabel, value, element.id(),
                                 expiredTime, removed);
                break;
            default:
                throw new AssertionError(String.format(
                          "Unknown index type '%s'", indexLabel.indexType()));
        }
    }
```

- 参数是索引id，数据HugeElement
- 先schema.getIndexLabel(ilId)，根据索引id获取到indexlabel
- 然后根据indexlabel中的字段获取element中的属性值
- 然后根据switch索引类型，来处理索引。


当用户的查询语义是：某属性值大于、小于、大于等于、小于等于、等于某个界限，或者属性值属于某个区间时，适合使用范围索引。比如：“年龄”、“价格”、“得分”等取值比较连续的属性。

范围索引处理方式如下：

- 先检查属性值个数是否为1，范围索引不支持组合索引。
- 然后updateIndex，保存索引

```java
			    E.checkState(propValues.size() == 1,
                             "Expect only one property in range index");
                Object value = NumericUtil.convertToNumber(propValues.get(0));
                this.updateIndex(indexLabel, value, element.id(),
                                 expiredTime, removed);
```
updateIndex 代码：

```java
private void updateIndex(IndexLabel indexLabel, Object propValue,
                             Id elementId, long expiredTime, boolean removed) {
        HugeIndex index = new HugeIndex(this.graph(), indexLabel);
        index.fieldValues(propValue);
        index.elementIds(elementId, expiredTime);

        if (removed) {
            this.doEliminate(this.serializer.writeIndex(index));
        } else {
            this.doAppend(this.serializer.writeIndex(index));
        }
    }
```

- 构造索引，根据removed来决定是append还是删除。
- 通过GraphSerializer序列化索引

这里我们来探索Serializer是如何做的，比如Binary:

```java
		    Id id = index.id();
            HugeType type = index.type();
            byte[] value = null;
            if (!type.isNumericIndex() && indexIdLengthExceedLimit(id)) {
                id = index.hashId();
                // Save field-values as column value if the key is a hash string
                value = StringEncoding.encode(index.fieldValues().toString());
            }

            entry = newBackendEntry(type, id);
            entry.column(this.formatIndexName(index), value);
            entry.subId(index.elementId());

            if (index.hasTtl()) {
                entry.ttl(index.ttl());
            }
```

- 生成一个BackendEntry，id为索引id
- column name 通过formatIndexName生成， value 一般为null
- subId为elementid

索引的id：

```java
public static Id formatIndexId(HugeType type, Id indexLabelId,
                                   Object fieldValues) {
        if (type.isStringIndex()) {
            String value = "";
            if (fieldValues instanceof Id) {
                value = IdGenerator.asStoredString((Id) fieldValues);
            } else if (fieldValues != null) {
                value = fieldValues.toString();
            }
            /*
             * Modify order between index label and field-values to put the
             * index label in front(hugegraph-1317)
             */
            String strIndexLabelId = IdGenerator.asStoredString(indexLabelId);
            return SplicingIdGenerator.splicing(strIndexLabelId, value);
        } else {
            assert type.isRangeIndex();
            int length = type.isRange4Index() ? 4 : 8;
            BytesBuffer buffer = BytesBuffer.allocate(4 + length);
            buffer.writeInt(SchemaElement.schemaId(indexLabelId));
            if (fieldValues != null) {
                E.checkState(fieldValues instanceof Number,
                             "Field value of range index must be number:" +
                             " %s", fieldValues.getClass().getSimpleName());
                byte[] bytes = number2bytes((Number) fieldValues);
                buffer.write(bytes);
            }
            return buffer.asId();
        }
    }
```

- 如果是rangeindex，id为 SchemaElement.schemaId(indexLabelId) + fieldValues
- 如果是字符串索引，id为 indexLabelId:fieldValues 拼接为字符串 （SplicingIdGenerator.splicing(）


```java
protected byte[] formatIndexName(HugeIndex index) {
        BytesBuffer buffer;
        Id elemId = index.elementId();
        if (!this.indexWithIdPrefix) {
            int idLen = 1 + elemId.length();
            buffer = BytesBuffer.allocate(idLen);
        } else {
            Id indexId = index.id();
            HugeType type = index.type();
            if (!type.isNumericIndex() && indexIdLengthExceedLimit(indexId)) {
                indexId = index.hashId();
            }
            int idLen = 1 + elemId.length() + 1 + indexId.length();
            buffer = BytesBuffer.allocate(idLen);
            // Write index-id
            buffer.writeIndexId(indexId, type);
        }
        // Write element-id
        buffer.writeId(elemId);
        // Write expired time if needed
        if (index.hasTtl()) {
            buffer.writeVLong(index.expiredTime());
        }

        return buffer.bytes();
    }
```

formatIndexName 决定了column name：

- 先写入indexId，也就是上面（formatIndexId）生成的index id
- 再写入elemId


最后写入存储后端时，

```java
 @Override
    public void insert(Session session, BackendEntry entry) {
        assert !entry.columns().isEmpty();
        for (BackendColumn col : entry.columns()) {
            assert entry.belongToMe(col) : entry;
            session.put(this.table(), col.name, col.value);
        }
    }
```

对于range 索引，key的前缀是Int的indexLabelId，中间是索引值的bytes，后缀是elementid，因此range索引天然是有序的。

存储结构：

	index_label_id | field_values | element_ids
	
对于二级索引，也是：

 	indexLabelId | fieldValues | element_ids
	
*   `field_values`: 属性的值，可以是单个属性，也可以是多个属性拼接而成
*   `index_label_id`: 索引标签的Id
*   `element_ids`: 顶点或边的Id

## 索引查询过程分析


查询要从GraphTransaction的query开始分析，针对ConditionQuery条件查询，会调用optimizeQueries优化查询。

```java

public QueryResults<BackendEntry> query(Query query) {
        if (!(query instanceof ConditionQuery)) {
            LOG.debug("Query{final:{}}", query);
            return super.query(query);
        }

        QueryList<BackendEntry> queries = this.optimizeQueries(query,
                                                               super::query);
        LOG.debug("{}", queries);
        return queries.empty() ? QueryResults.empty() :
                                 queries.fetch(this.pageSize);
    }
```

optimizeQueries 会将condtion query flatten展开（比如in查询，展开成多个查询），然后针对每个cq做查询。

针对每个cq，会调用indexQuery走索引查询。

```java
protected <R> QueryList<R> optimizeQueries(Query query,
                                             QueryResults.Fetcher<R> fetcher) {
        QueryList<R> queries = new QueryList<>(query, fetcher);
        for (ConditionQuery cq: ConditionQueryFlatten.flatten(
                                (ConditionQuery) query)) {
            // Optimize by sysprop
            Query q = this.optimizeQuery(cq);
            /*
             * NOTE: There are two possibilities for this query:
             * 1.sysprop-query, which would not be empty.
             * 2.index-query result(ids after optimization), which may be empty.
             */
            if (q == null) {
                queries.add(this.indexQuery(cq), this.batchSize);
            } else if (!q.empty()) {
                queries.add(q);
            }
        }
        return queries;
    }
```

索引查询，核心代码在 GraphIndexTransaction.queryIndex

```java
@Watched(prefix = "index")
    public IdHolderList queryIndex(ConditionQuery query) {
        // Index query must have been flattened in Graph tx
        query.checkFlattened();

        // NOTE: Currently we can't support filter changes in memory
        if (this.hasUpdate()) {
            throw new HugeException("Can't do index query when " +
                                    "there are changes in transaction");
        }

        // Can't query by index and by non-label sysprop at the same time
        List<Condition> conds = query.syspropConditions();
        if (conds.size() > 1 ||
            (conds.size() == 1 && !query.containsCondition(HugeKeys.LABEL))) {
            throw new HugeException("Can't do index query with %s and %s",
                                    conds, query.userpropConditions());
        }

        // Query by index
        query.optimized(OptimizedType.INDEX);
        if (query.allSysprop() && conds.size() == 1 &&
            query.containsCondition(HugeKeys.LABEL)) {
            // Query only by label
            return this.queryByLabel(query);
        } else {
            // Query by userprops (or userprops + label)
            return this.queryByUserprop(query);
        }
    }
```

会先做一些检查，然后判断是否有属性条件，如果没有则直接查询对应label，否则走queryByUserprop，根据属性值查询结果。

``` java
@Watched(prefix = "index")
    private IdHolderList queryByUserprop(ConditionQuery query) {
        // Get user applied label or collect all qualified labels with
        // related index labels
        Set<MatchedIndex> indexes = this.collectMatchedIndexes(query);
        if (indexes.isEmpty()) {
            Id label = query.condition(HugeKeys.LABEL);
            throw noIndexException(this.graph(), query, label);
        }

        // Value type of Condition not matched
        boolean paging = query.paging();
        if (!validQueryConditionValues(this.graph(), query)) {
            return IdHolderList.empty(paging);
        }

        // Do index query
        IdHolderList holders = new IdHolderList(paging);
        for (MatchedIndex index : indexes) {
            for (IndexLabel il : index.indexLabels()) {
                validateIndexLabel(il);
            }
            if (paging && index.indexLabels().size() > 1) {
                throw new NotSupportException("joint index query in paging");
            }

            if (index.containsSearchIndex()) {
                // Do search-index query
                holders.addAll(this.doSearchIndex(query, index));
            } else {
                // Do secondary-index, range-index or shard-index query
                IndexQueries queries = index.constructIndexQueries(query);
                assert !paging || queries.size() <= 1;
                IdHolder holder = this.doSingleOrJointIndex(queries);
                holders.add(holder);
            }

            /*
             * NOTE: need to skip the offset if offset > 0, but can't handle
             * it here because the query may a sub-query after flatten,
             * so the offset will be handle in QueryList.IndexQuery
             *
             * TODO: finish early here if records exceeds required limit with
             *       FixedIdHolder.
             */
        }
        return holders;
    }
```

queryByUserprop 会先查询出匹配的索引(collectMatchedIndexes)，如果没匹配到索引，就会报错。

如果匹配到多个索引，依次查询，如果是search索引，走doSearchIndex，反之先constructIndexQueries，然后doSingleOrJointIndex。

### 搜索索引

搜索索引，之所以特殊处理，因为要分词：

```java
@Watched(prefix = "index")
    private IdHolderList doSearchIndex(ConditionQuery query,
                                       MatchedIndex index) {
        query = this.constructSearchQuery(query, index);
        // Sorted by matched count
        IdHolderList holders = new SortByCountIdHolderList(query.paging());
        List<ConditionQuery> flatten = ConditionQueryFlatten.flatten(query);
        for (ConditionQuery q : flatten) {
            if (!q.noLimit() && flatten.size() > 1) {
                // Increase limit for union operation
                increaseLimit(q);
            }
            IndexQueries queries = index.constructIndexQueries(q);
            assert !query.paging() || queries.size() <= 1;
            IdHolder holder = this.doSingleOrJointIndex(queries);
            // NOTE: ids will be merged into one IdHolder if not in paging
            holders.add(holder);
        }
        return holders;
    }
```

- 先构造查询，然后组合结果
- 重点是如何构造查询的

``` java
private ConditionQuery constructSearchQuery(ConditionQuery query,
                                                MatchedIndex index) {
        ConditionQuery originQuery = query;
        Set<Id> indexFields = new HashSet<>();
        // Convert has(key, text) to has(key, textContainsAny(word1, word2))
        for (IndexLabel il : index.indexLabels()) {
            if (il.indexType() != IndexType.SEARCH) {
                continue;
            }
            Id indexField = il.indexField();
            String fieldValue = (String) query.userpropValue(indexField);
            Set<String> words = this.segmentWords(fieldValue);
            indexFields.add(indexField);

            query = query.copy();
            query.unsetCondition(indexField);
            query.query(Condition.textContainsAny(indexField, words));
        }

        // Register results filter to compare property value and search text
        query.registerResultsFilter(elem -> {
            for (Condition cond : originQuery.conditions()) {
                Object key = cond.isRelation() ? ((Relation) cond).key() : null;
                if (key instanceof Id && indexFields.contains(key)) {
                    // This is an index field of search index
                    Id field = (Id) key;
                    assert elem != null;
                    HugeProperty<?> property = elem.getProperty(field);
                    String propValue = propertyValueToString(property.value());
                    String fieldValue = (String) originQuery.userpropValue(field);
                    if (this.matchSearchIndexWords(propValue, fieldValue)) {
                        continue;
                    }
                    return false;
                }
                if (!cond.test(elem)) {
                    return false;
                }
            }
            return true;
        });

        return query;
    }
```

- 先分词
- 然后resetquery，Convert has(key, text) to has(key, textContainsAny(word1, word2))
- 最后，索引查询可能匹配到多个结果，registerResultsFilter 注册一个结果过滤器，对结果做过滤


### 普通索引

普通索引，也是先构造索引查询：

``` java
ublic IndexQueries constructIndexQueries(ConditionQuery query) {
            // Condition query => Index Queries
            if (this.indexLabels().size() == 1) {
                /*
                 * Query by single index or composite index
                 */
                IndexLabel il = this.indexLabels().iterator().next();
                ConditionQuery indexQuery = constructQuery(query, il);
                assert indexQuery != null;
                return IndexQueries.of(il, indexQuery);
            } else {
                /*
                 * Query by joint indexes
                 */
                IndexQueries queries = buildJointIndexesQueries(query, this);
                assert !queries.isEmpty();
                return queries;
            }
        }
```

如果只匹配到一个索引，直接走这个索引，最简单的情况，

如果匹配到多个索引，这个时候要走联合查询了（buildJointIndexesQueries）

最后，通过doSingleOrJointIndex来获取结果：

``` java
    @Watched(prefix = "index")
    private IdHolder doSingleOrJointIndex(IndexQueries queries) {
        if (queries.size() == 1) {
            return this.doSingleOrCompositeIndex(queries);
        } else {
            return this.doJointIndex(queries);
        }
    }
```

如果queries.size > 1，代表要走联合索引。但是一般db一次查询通常直走一个索引，hugegraph也差不多：

``` java
@Watched(prefix = "index")
    private IdHolder doJointIndex(IndexQueries queries) {
        if (queries.oomRisk()) {
            LOG.warn("There is OOM risk if the joint operation is based on a " +
                     "large amount of data, please use single index + filter " +
                     "instead of joint index: {}", queries.rootQuery());
        }
        // All queries are joined with AND
        Set<Id> intersectIds = null;
        boolean filtering = false;
        IdHolder resultHolder = null;
        for (Map.Entry<IndexLabel, ConditionQuery> e : queries.entrySet()) {
            IndexLabel indexLabel = e.getKey();
            ConditionQuery query = e.getValue();
            assert !query.paging();
            if (!query.noLimit() && queries.size() > 1) {
                // Unset limit for intersection operation
                query.limit(Query.NO_LIMIT);
            }
            /*
             * Try to query by joint indexes:
             * 1 If there is any index exceeded the threshold, transform into
             *   partial index query, then filter after back-table.
             * 1.1 Return the holder of the first index that not exceeded the
             *     threshold if there exists one index, this holder will be used
             *     as the only query condition.
             * 1.2 Return the holder of the first index if all indexes exceeded
             *     the threshold.
             * 2 Else intersect holders for all indexes, and return intersection
             *   ids of all indexes.
             */
            IdHolder holder = this.doIndexQuery(indexLabel, query);
            if (resultHolder == null) {
                resultHolder = holder;
            }
            assert this.indexIntersectThresh > 0; // default value is 1000
            Set<Id> ids = ((BatchIdHolder) holder).peekNext(
                          this.indexIntersectThresh).ids();
            if (ids.size() >= this.indexIntersectThresh) {
                // Transform into filtering
                filtering = true;
                query.optimized(OptimizedType.INDEX_FILTER);
            } else if (filtering) {
                assert ids.size() < this.indexIntersectThresh;
                resultHolder = holder;
                break;
            } else {
                if (intersectIds == null) {
                    intersectIds = ids;
                } else {
                    CollectionUtil.intersectWithModify(intersectIds, ids);
                }
                if (intersectIds.isEmpty()) {
                    break;
                }
            }
        }

        if (filtering) {
            return resultHolder;
        } else {
            assert intersectIds != null;
            return new FixedIdHolder(queries.asJointQuery(), intersectIds);
        }
    }
```

- 依次读取，先读取indexIntersectThresh 个数的匹配索引id，indexIntersectThresh用来控制1次读取索引id的个数，这个默认是1000,
- 如果地个数》=indexIntersectThresh，这个时候hugegraph认为匹配结果数太多了，不能直接走索引查询到结果，需要走过滤（OptimizedType.INDEX_FILTER），也就是读取可能的候选结果，然后通过查询条件过滤结果。
- 如果有一个索引较小，resultHolder缓存较小索引的
- 如果几个索引都小于indexIntersectThresh，这是最理想情况，直接取ids的交集（CollectionUtil.intersectWithModify）

读取到id后，就是根据id读取结果，过滤结果了。

## 如何通过索引读取到匹配的id？

关键代码在AbstractTransaction：

```java
@Watched(prefix = "tx")
    public QueryResults<BackendEntry> query(Query query) {
        LOG.debug("Transaction query: {}", query);
        /*
         * NOTE: it's dangerous if an IdQuery/ConditionQuery is empty
         * check if the query is empty and its class is not the Query itself
         */
        if (query.empty() && !query.getClass().equals(Query.class)) {
            throw new BackendException("Query without any id or condition");
        }

        Query squery = this.serializer.writeQuery(query);

        // Do rate limit if needed
        RateLimiter rateLimiter = this.graph.readRateLimiter();
        if (rateLimiter != null && query.resultType().isGraph()) {
            double time = rateLimiter.acquire(1);
            if (time > 0) {
                LOG.debug("Waited for {}s to query", time);
            }
            BackendEntryIterator.checkInterrupted();
        }

        this.beforeRead();
        try {
            return new QueryResults<>(this.store.query(squery), query);
        } finally {
            this.afterRead(); // TODO: not complete the iteration currently
        }
    }
```

逐级往下，核心代码在writeQueryCondition：

``` javascript
	@Override
    protected Query writeQueryCondition(Query query) {
        HugeType type = query.resultType();
        if (!type.isIndex()) {
            return query;
        }

        ConditionQuery cq = (ConditionQuery) query;

        if (type.isNumericIndex()) {
            // Convert range-index/shard-index query to id range query
            return this.writeRangeIndexQuery(cq);
        } else {
            assert type.isSearchIndex() || type.isSecondaryIndex() ||
                   type.isUniqueIndex();
            // Convert secondary-index or search-index query to id query
            return this.writeStringIndexQuery(cq);
        }
    }
```

如果是rangeindex 索引，会转换为scan ` indexlabelid:start - indexlabelid:end` 的查询

``` javascript
private Query writeRangeIndexQuery(ConditionQuery query) {
        Id index = query.condition(HugeKeys.INDEX_LABEL_ID);
        E.checkArgument(index != null, "Please specify the index label");

        List<Condition> fields = query.syspropConditions(HugeKeys.FIELD_VALUES);
        E.checkArgument(!fields.isEmpty(),
                        "Please specify the index field values");

        HugeType type = query.resultType();
        Id start = null;
        if (query.paging() && !query.page().isEmpty()) {
            byte[] position = PageState.fromString(query.page()).position();
            start = new BinaryId(position, null);
        }

        RangeConditions range = new RangeConditions(fields);
        if (range.keyEq() != null) {
            Id id = formatIndexId(type, index, range.keyEq(), true);
            if (start == null) {
                return new IdPrefixQuery(query, id);
            }
            E.checkArgument(Bytes.compare(start.asBytes(), id.asBytes()) >= 0,
                            "Invalid page out of lower bound");
            return new IdPrefixQuery(query, start, id);
        }

        Object keyMin = range.keyMin();
        Object keyMax = range.keyMax();
        boolean keyMinEq = range.keyMinEq();
        boolean keyMaxEq = range.keyMaxEq();
        if (keyMin == null) {
            E.checkArgument(keyMax != null,
                            "Please specify at least one condition");
            // Set keyMin to min value
            keyMin = NumericUtil.minValueOf(keyMax.getClass());
            keyMinEq = true;
        }

        Id min = formatIndexId(type, index, keyMin, false);
        if (!keyMinEq) {
            /*
             * Increase 1 to keyMin, index GT query is a scan with GT prefix,
             * inclusiveStart=false will also match index started with keyMin
             */
            increaseOne(min.asBytes());
            keyMinEq = true;
        }

        if (start == null) {
            start = min;
        } else {
            E.checkArgument(Bytes.compare(start.asBytes(), min.asBytes()) >= 0,
                            "Invalid page out of lower bound");
        }

        if (keyMax == null) {
            keyMax = NumericUtil.maxValueOf(keyMin.getClass());
            keyMaxEq = true;
        }
        Id max = formatIndexId(type, index, keyMax, false);
        if (keyMaxEq) {
            keyMaxEq = false;
            increaseOne(max.asBytes());
        }
        return new IdRangeQuery(query, start, keyMinEq, max, keyMaxEq);
    }
```

如果是其他索引，则转换为前缀匹配查询:

``` java
private Query writeStringIndexQuery(ConditionQuery query) {
        E.checkArgument(query.allSysprop() &&
                        query.conditions().size() == 2,
                        "There should be two conditions: " +
                        "INDEX_LABEL_ID and FIELD_VALUES" +
                        "in secondary index query");

        Id index = query.condition(HugeKeys.INDEX_LABEL_ID);
        Object key = query.condition(HugeKeys.FIELD_VALUES);

        E.checkArgument(index != null, "Please specify the index label");
        E.checkArgument(key != null, "Please specify the index key");

        Id prefix = formatIndexId(query.resultType(), index, key, true);
        return prefixQuery(query, prefix);
    }
```

查询到rocksdb后端的时候：

```java
protected BackendColumnIterator queryBy(Session session, Query query) {
        // Query all
        if (query.empty()) {
            return this.queryAll(session, query);
        }

        // Query by prefix
        if (query instanceof IdPrefixQuery) {
            IdPrefixQuery pq = (IdPrefixQuery) query;
            return this.queryByPrefix(session, pq);
        }

        // Query by range
        if (query instanceof IdRangeQuery) {
            IdRangeQuery rq = (IdRangeQuery) query;
            return this.queryByRange(session, rq);
        }

        // Query by id
        if (query.conditions().isEmpty()) {
            assert !query.ids().isEmpty();
            // NOTE: this will lead to lazy create rocksdb iterator
            return new BackendColumnIteratorWrapper(new FlatMapperIterator<>(
                   query.ids().iterator(), id -> this.queryById(session, id)
            ));
        }

        // Query by condition (or condition + id)
        ConditionQuery cq = (ConditionQuery) query;
        return this.queryByCond(session, cq);
    }
```

前缀查询：

``` java
protected BackendColumnIterator queryByPrefix(Session session,
                                                  IdPrefixQuery query) {
        int type = query.inclusiveStart() ?
                   Session.SCAN_GTE_BEGIN : Session.SCAN_GT_BEGIN;
        type |= Session.SCAN_PREFIX_END;
        return session.scan(this.table(), query.start().asBytes(),
                            query.prefix().asBytes(), type);
    }
```

range查询：

``` java
protected BackendColumnIterator queryByRange(Session session,
                                                 IdRangeQuery query) {
        byte[] start = query.start().asBytes();
        byte[] end = query.end() == null ? null : query.end().asBytes();
        int type = query.inclusiveStart() ?
                   Session.SCAN_GTE_BEGIN : Session.SCAN_GT_BEGIN;
        if (end != null) {
            type |= query.inclusiveEnd() ?
                    Session.SCAN_LTE_END : Session.SCAN_LT_END;
        }
        return session.scan(this.table(), start, end, type);
    }
```

查询后，在BinarySerializer中，通过readIndex还原为index：

``` java
	@Override
    public HugeIndex readIndex(HugeGraph graph, ConditionQuery query,
                               BackendEntry bytesEntry) {
        if (bytesEntry == null) {
            return null;
        }

        BinaryBackendEntry entry = this.convertEntry(bytesEntry);
        // NOTE: index id without length prefix
        byte[] bytes = entry.id().asBytes();
        HugeIndex index = HugeIndex.parseIndexId(graph, entry.type(), bytes);

        Object fieldValues = null;
        if (!index.type().isRangeIndex()) {
            fieldValues = query.condition(HugeKeys.FIELD_VALUES);
            if (!index.fieldValues().equals(fieldValues)) {
                // Update field-values for hashed or encoded index-id
                index.fieldValues(fieldValues);
            }
        }

        this.parseIndexName(graph, query, entry, index, fieldValues);
        return index;
    }
```

parseIndexId 和parseIndexName 是存储的decode操作，代码类似，一个存，一个读。

## 索引与全局排序优化

这里提一个问题，要对符合条件的结果做全局排序怎么优化？

比如，我们需要按更新时间(update_time）排序，当没有其他条件时，可以将排序转换为update_time>0 的查询，因为range索引默认是有序的，从小到大（详见上面的存储结构分析）。

如果要倒序怎么办？
  
  - 业务简单时，可以冗余一个字段，比如update_time_desc，取一个固定值-update_time, 这样最新的的数据在前面。

但是，这种查询，在有其他条件时就无效了，详见doJointIndex，这种情况如何优化了？

我们下期再聊。



--------------------------

感谢您的认真阅读。

如果你觉得有帮助，欢迎点赞支持！

不定期分享软件开发经验，欢迎关注作者, 一起交流软件开发：

