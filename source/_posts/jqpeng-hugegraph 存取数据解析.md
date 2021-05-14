---
title: hugegraph 存取数据解析
tags: ["hugegraph","知识图谱","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-12-09 11:58
---
文章作者:jqpeng
原文链接: [hugegraph 存取数据解析](https://www.cnblogs.com/xiaoqi/p/hugegraph-storage.html)

hugegraph 是百度开源的图数据库，支持hbase，mysql，rocksdb等作为存储后端。本文以EDGE 存储，hbase为存储后端，来探索hugegraph是如何存取数据的。

## 存数据

### 序列化

![Edge](https://gitee.com/jadepeng/pic/raw/master/pic/2020/12/9/1607480289186.png)

首先需要序列化，`hbase` 使用BinarySerializer:

- keyWithIdPrefix 和indexWithIdPrefix都是false


这个后面会用到。


    public class HbaseSerializer extends BinarySerializer {
    
        public HbaseSerializer() {
            super(false, true);
        }
    }


要存到db，首先需要序列化为BackendEntry，`BackendEntry` 是图数据库和后端存储的传输对象，Hbase对应的是`BinaryBackendEntry`:


    public class BinaryBackendEntry implements BackendEntry {
    
        private static final byte[] EMPTY_BYTES = new byte[]{};
    
        private final HugeType type;
        private final BinaryId id;
        private Id subId;
        private final List<BackendColumn> columns;
        private long ttl;
    
        public BinaryBackendEntry(HugeType type, byte[] bytes) {
            this(type, BytesBuffer.wrap(bytes).parseId(type));
        }
    
        public BinaryBackendEntry(HugeType type, BinaryId id) {
            this.type = type;
            this.id = id;
            this.subId = null;
            this.columns = new ArrayList<>();
            this.ttl = 0L;
        }


我们来看序列化，序列化，其实就是要将数据放到entry的column列里。

- `hbase` 的`keyWithIdPrefix`是false，因此`name`不包含ownerVertexId（参考下面的EdgeId，去掉ownerVertexId）



     public BackendEntry writeEdge(HugeEdge edge) {
            BinaryBackendEntry entry = newBackendEntry(edge);
            byte[] name = this.keyWithIdPrefix ?
                          this.formatEdgeName(edge) : EMPTY_BYTES;
            byte[] value = this.formatEdgeValue(edge);
            entry.column(name, value);
    
            if (edge.hasTtl()) {
                entry.ttl(edge.ttl());
            }
    
            return entry;
        }


EdgeId：


        private final Id ownerVertexId;
        private final Directions direction;
        private final Id edgeLabelId;
        private final String sortValues;
        private final Id otherVertexId;
    
        private final boolean directed;
        private String cache;
    


### backend 存储

生成BackendEntry后，通过store机制，交给后端的backend存储。

EDGE的保存，对应HbaseTables.Edge:


    public static class Edge extends HbaseTable {
    
            @Override
            public void insert(Session session, BackendEntry entry) {
                long ttl = entry.ttl();
                if (ttl == 0L) {
                    session.put(this.table(), CF, entry.id().asBytes(),
                                entry.columns());
                } else {
                    session.put(this.table(), CF, entry.id().asBytes(),
                                entry.columns(), ttl);
                }
            }
    }


CF 是固定的f：


        protected static final byte[] CF = "f".getBytes();


`session.put` 对应：


     @Override
            public void put(String table, byte[] family, byte[] rowkey,
                            Collection<BackendColumn> columns) {
                Put put = new Put(rowkey);
                for (BackendColumn column : columns) {
                    put.addColumn(family, column.name, column.value);
                }
                this.batch(table, put);
            }


可以看出，存储时，edgeid作为`rowkey`，然后把去除`ownerVertexId`后的`edgeid`作为`column.name`

## EDGE 读取

### 从backend读取BackendEntry

读取就是从hbase读取result，转换为BinaryBackendEntry，再转成Edge。

读取，是scan的过程：


     /**
             * Inner scan: send scan request to HBase and get iterator
             */
            @Override
            public RowIterator scan(String table, Scan scan) {
                assert !this.hasChanges();
    
                try (Table htable = table(table)) {
                    return new RowIterator(htable.getScanner(scan));
                } catch (IOException e) {
                    throw new BackendException(e);
                }
            }


scan后，返回`BackendEntryIterator`


    protected BackendEntryIterator newEntryIterator(Query query,
                                                        RowIterator rows) {
            return new BinaryEntryIterator<>(rows, query, (entry, row) -> {
                E.checkState(!row.isEmpty(), "Can't parse empty HBase result");
                byte[] id = row.getRow();
                if (entry == null || !Bytes.prefixWith(id, entry.id().asBytes())) {
                    HugeType type = query.resultType();
                    // NOTE: only support BinaryBackendEntry currently
                    entry = new BinaryBackendEntry(type, id);
                }
                try {
                    this.parseRowColumns(row, entry, query);
                } catch (IOException e) {
                    throw new BackendException("Failed to read HBase columns", e);
                }
                return entry;
            });
        }


注意，`new BinaryBackendEntry(type, id)` 时，BinaryBackendEntry的id并不是`rowkey`，而是对rowkey做了处理：


    public BinaryId parseId(HugeType type) {
            if (type.isIndex()) {
                return this.readIndexId(type);
            }
            // Parse id from bytes
            int start = this.buffer.position();
            /*
             * Since edge id in edges table doesn't prefix with leading 0x7e,
             * so readId() will return the source vertex id instead of edge id,
             * can't call: type.isEdge() ? this.readEdgeId() : this.readId();
             */
            Id id = this.readId();
            int end = this.buffer.position();
            int len = end - start;
            byte[] bytes = new byte[len];
            System.arraycopy(this.array(), start, bytes, 0, len);
            return new BinaryId(bytes, id);
        }


这里是先读取ownervertexId作为Id部分, 然后将剩余的直接放入bytes，组合成BinaryId，和序列化的时候有差别，为什么这么设计呢？原来不管是vertex还是edge，都是当成Vertex来读取的。


    protected final BinaryBackendEntry newBackendEntry(HugeEdge edge) {
            BinaryId id = new BinaryId(formatEdgeName(edge),
                                       edge.idWithDirection());
            return newBackendEntry(edge.type(), id);
        }
    
    public EdgeId directed(boolean directed) {
        return new EdgeId(this.ownerVertexId, this.direction, this.edgeLabelId,
                          this.sortValues, this.otherVertexId, directed);
    }


序列化的时候是`EdgeId`。

`BackendEntryIterator`迭代器支持对结果进行merge, 上面代码里的`!Bytes.prefixWith(id, entry.id().asBytes()))` 就是对比是否是同一个ownervertex，如果是同一个，则放到同一个BackendEntry的Columns里。


         public BinaryEntryIterator(BackendIterator<Elem> results, Query query,
                                   BiFunction<BackendEntry, Elem, BackendEntry> m)
    
        @Override
        protected final boolean fetch() {
            assert this.current == null;
            if (this.next != null) {
                this.current = this.next;
                this.next = null;
            }
    
            while (this.results.hasNext()) {
                Elem elem = this.results.next();
                BackendEntry merged = this.merger.apply(this.current, elem);
                E.checkState(merged != null, "Error when merging entry");
                if (this.current == null) {
                    // The first time to read
                    this.current = merged;
                } else if (merged == this.current) {
                    // The next entry belongs to the current entry
                    assert this.current != null;
                    if (this.sizeOf(this.current) >= INLINE_BATCH_SIZE) {
                        break;
                    }
                } else {
                    // New entry
                    assert this.next == null;
                    this.next = merged;
                    break;
                }
    
                // When limit exceed, stop fetching
                if (this.reachLimit(this.fetched() - 1)) {
                    // Need remove last one because fetched limit + 1 records
                    this.removeLastRecord();
                    this.results.close();
                    break;
                }
            }
    
            return this.current != null;
        }
    


### 从BackendEntry转换为edge

然后再来看读取数据`readVertex`，前面说了，就算是edge，其实也是当vertex来读取的：


     @Override
        public HugeVertex readVertex(HugeGraph graph, BackendEntry bytesEntry) {
            if (bytesEntry == null) {
                return null;
            }
            BinaryBackendEntry entry = this.convertEntry(bytesEntry);
    
            // Parse id
            Id id = entry.id().origin();
            Id vid = id.edge() ? ((EdgeId) id).ownerVertexId() : id;
            HugeVertex vertex = new HugeVertex(graph, vid, VertexLabel.NONE);
    
            // Parse all properties and edges of a Vertex
            for (BackendColumn col : entry.columns()) {
                if (entry.type().isEdge()) {
                    // NOTE: the entry id type is vertex even if entry type is edge
                    // Parse vertex edges
                    this.parseColumn(col, vertex);
                } else {
                    assert entry.type().isVertex();
                    // Parse vertex properties
                    assert entry.columnsSize() == 1 : entry.columnsSize();
                    this.parseVertex(col.value, vertex);
                }
            }
    
            return vertex;
        }


逻辑：

- 先读取ownervertexid，生成HugeVertex，这个时候只知道id，不知道vertexlabel，所以设置为VertexLabel.NONE
- 然后，读取BackendColumn，一个edge，一个Column（name是edgeid去除ownervertexid后的部分，value是边数据）


读取是在`parseColumn`:


    protected void parseColumn(BackendColumn col, HugeVertex vertex) {
            BytesBuffer buffer = BytesBuffer.wrap(col.name);
            Id id = this.keyWithIdPrefix ? buffer.readId() : vertex.id();
            E.checkState(buffer.remaining() > 0, "Missing column type");
            byte type = buffer.read();
            // Parse property
            if (type == HugeType.PROPERTY.code()) {
                Id pkeyId = buffer.readId();
                this.parseProperty(pkeyId, BytesBuffer.wrap(col.value), vertex);
            }
            // Parse edge
            else if (type == HugeType.EDGE_IN.code() ||
                     type == HugeType.EDGE_OUT.code()) {
                this.parseEdge(col, vertex, vertex.graph());
            }
            // Parse system property
            else if (type == HugeType.SYS_PROPERTY.code()) {
                // pass
            }
            // Invalid entry
            else {
                E.checkState(false, "Invalid entry(%s) with unknown type(%s): 0x%s",
                             id, type & 0xff, Bytes.toHex(col.name));
            }
        }


从``col.name`读取type，如果是edge，则parseEdge:


    protected void parseEdge(BackendColumn col, HugeVertex vertex,
                                 HugeGraph graph) {
            // owner-vertex + dir + edge-label + sort-values + other-vertex
    
            BytesBuffer buffer = BytesBuffer.wrap(col.name);
            if (this.keyWithIdPrefix) {
                // Consume owner-vertex id
                buffer.readId();
            }
            byte type = buffer.read();
            Id labelId = buffer.readId();
            String sortValues = buffer.readStringWithEnding();
            Id otherVertexId = buffer.readId();
    
            boolean direction = EdgeId.isOutDirectionFromCode(type);
            EdgeLabel edgeLabel = graph.edgeLabelOrNone(labelId);
    
            // Construct edge
            HugeEdge edge = HugeEdge.constructEdge(vertex, direction, edgeLabel,
                                                   sortValues, otherVertexId);
    
            // Parse edge-id + edge-properties
            buffer = BytesBuffer.wrap(col.value);
    
            //Id id = buffer.readId();
    
            // Parse edge properties
            this.parseProperties(buffer, edge);
    
            // Parse edge expired time if needed
            if (edge.hasTtl()) {
                this.parseExpiredTime(buffer, edge);
            }
        }


从col.name依次读取出type,labelId,sortValues和otherVertexId：


            byte type = buffer.read();
            Id labelId = buffer.readId();
            String sortValues = buffer.readStringWithEnding();
            Id otherVertexId = buffer.readId();


然后根据labelid找到  `EdgeLabel edgeLabel = graph.edgeLabelOrNone(labelId);`

创建`edge`, 解析边属性`parseProperties`

最后读取`Ttl`, 处理结果的时候，会过滤过期数据。

