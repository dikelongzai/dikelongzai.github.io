---
title: 使用 Spark SQL 高效地读写 HBase
categories:
  - 大数据
  - 计算框架
  
tags:
  - spark
abbrlink: 33760
date: 2019-03-03 01:29:56
---
Apache Spark 和 Apache HBase 是两个使用比较广泛的大数据组件。很多场景需要使用 Spark 分析/查询 HBase 中的数据，而目前 Spark 内置是支持很多数据源的，其中就包括了 HBase，但是内置的读取数据源还是使用了 TableInputFormat 来读取 HBase 中的数据。这个 TableInputFormat 有一些缺点：

一个 Task 里面只能启动一个 Scan 去 HBase 中读取数据；
TableInputFormat 中不支持 BulkGet；
不能享受到 Spark SQL 内置的 catalyst 引擎的优化。
基于这些问题，来自 Hortonworks 的工程师们为我们带来了全新的 Apache Spark—Apache HBase Connector，下面简称 SHC。通过这个类库，我们可以直接使用 Spark SQL 将 DataFrame 中的数据写入到 HBase 中；而且我们也可以使用 Spark SQL 去查询 HBase 中的数据，在查询 HBase 的时候充分利用了 catalyst 引擎做了许多优化，比如分区修剪（partition pruning），列修剪（column pruning），谓词下推（predicate pushdown）和数据本地性（data locality）等等。因为有了这些优化，通过 Spark 查询 HBase 的速度有了很大的提升。

注意：SHC 同时还提供了将 DataFrame 中的数据直接写入到 HBase 中，但是整个代码并没有什么优化的地方，所以本文对这部分不进行介绍。感兴趣的读者可以直接到这里查看相关写数据到 HBase 的代码。

文章目录
+ 1 SHC 是如何实现查询优化的呢
* 1.1 将使用 Rowkey 的查询转换成 get 查询
+ 2 组合 RowKey 的查询优化
+ 3 scan 查询优化
SHC 是如何实现查询优化的呢
SHC 主要使用下面的几种优化，使得 Spark 获取 HBase 的数据扫描范围得到减少，提高了数据读取的效率。

将使用 Rowkey 的查询转换成 get 查询
我们都知道，HBase 中使用 Get 查询的效率是非常高的，所以如果查询的过滤条件是针对 RowKey 进行的，那么我们可以将它转换成 Get 查询。为了说明这点，我们使用下面的例子进行说明。假设我们定义好的 HBase catalog 如下：
```
val catalog = s"""{
  |"table":{"namespace":"default", "name":"iteblog", "tableCoder":"PrimitiveType"},
  |"rowkey":"key",
  |"columns":{
    |"col0":{"cf":"rowkey", "col":"id", "type":"int"},
    |"col1":{"cf":"cf1", "col":"col1", "type":"boolean"},
    |"col2":{"cf":"cf2", "col":"col2", "type":"double"},
    |"col3":{"cf":"cf3", "col":"col3", "type":"float"},
    |"col4":{"cf":"cf4", "col":"col4", "type":"int"},
    |"col5":{"cf":"cf5", "col":"col5", "type":"bigint"},
    |"col6":{"cf":"cf6", "col":"col6", "type":"smallint"},
    |"col7":{"cf":"cf7", "col":"col7", "type":"string"},
    |"col8":{"cf":"cf8", "col":"col8", "type":"tinyint"}
  |}
|}""".stripMargin
```
那么如果有类似下面的查询
```
val df = withCatalog(catalog)
df.createOrReplaceTempView("iteblog_table")
sqlContext.sql("select * from iteblog_table where id = 1")
sqlContext.sql("select * from iteblog_table where id = 1 or id = 2")
sqlContext.sql("select * from iteblog_table where id in (1, 2)")
```
因为查询条件直接是针对 RowKey 进行的，所以这种情况直接可以转换成 Get 或者 BulkGet 请求的。第一个 SQL 查询过程类似于下面过程
![sparkshc1](/public/image/spark-shc-1.png)
SHC：使用 Spark SQL 高效地读写 HBase
如果想及时了解Spark、Hadoop或者Hbase相关的文章，欢迎关注微信公共帐号：iteblog_hadoop
后面两条 SQL 查询其实是等效的，在实现上会把 key in (x1, x2, x3..) 转换成 (key == x1) or (key == x2) or ... 的。整个查询流程如下：
![sparkshc2](/public/image/spark-shc-2.png)
SHC：使用 Spark SQL 高效地读写 HBase
如果想及时了解Spark、Hadoop或者Hbase相关的文章，欢迎关注微信公共帐号：iteblog_hadoop
如果我们的查询里面有 Rowkey 还有其他列的过滤，比如下面的例子
```
sqlContext.sql("select id, col6, col8 from iteblog_table where id = 1 and col7 = 'xxx'")
```
那么上面的 SQL 翻译成 HBase 的下面查询
```
val filter = new SingleColumnValueFilter(
              Bytes.toBytes("cf7"), Bytes.toBytes("col7 ")
              CompareOp.EQUAL,
             Bytes.toBytes("xxx"))
 
val g = new Get(Bytes.toBytes(1))
g.addColumn(Bytes.toBytes("cf6"), Bytes.toBytes("col6"))
g.addColumn(Bytes.toBytes("cf8"), Bytes.toBytes("col8"))
g.setFilter(filter)
```
如果有多个 and 条件，都是使用 SingleColumnValueFilter 进行过滤的，这个都好理解。如果我们有下面的查询
```
sqlContext.sql("select id, col6, col8 from iteblog_table where id = 1 or col7 = 'xxx'")
```
那么在 shc 里面是怎么进行的呢？事实上，如果碰到非 RowKey 的过滤，那么这种查询是需要扫描 HBase 的全表的。上面的查询在 shc 里面就是将 HBase 里面的所有数据拿到，然后传输到 Spark ，再通过 Spark 里面进行过滤，可见 shc 在这种情况下效率是很低下的。

注意，上面的查询在 shc 返回的结果是错误的。具体原因是在将 id = 1 or col7 = 'xxx' 查询条件进行合并时，丢弃了所有的查找条件，相当于返回表的所有数据。定位到代码可以参见下面的
```
def or[T](left: HRF[T],
            right: HRF[T])(implicit ordering: Ordering[T]): HRF[T] = {
    val ranges = ScanRange.or(left.ranges, right.ranges)
    val typeFilter = TypedFilter.or(left.tf, right.tf)
    HRF(ranges, typeFilter, left.handled && right.handled)
}
```
同理，类似于下面的查询在 shc 里面其实都是全表扫描，并且将所有的数据返回到 Spark 层面上再进行一次过滤。
```
sqlContext.sql("select id, col6, col8 from iteblog_table where id = 1 or col7 <= 'xxx'")
sqlContext.sql("select id, col6, col8 from iteblog_table where id = 1 or col7 >= 'xxx'")
sqlContext.sql("select id, col6, col8 from iteblog_table where col7 = 'xxx'")
```
很显然，这种方式查询效率并不高，一种可行的方案是将算子下推到 HBase 层面，在 HBase 层面通过 SingleColumnValueFilter 过滤一部分数据，然后再返回到 Spark，这样可以节省很多数据的传输。

组合 RowKey 的查询优化
shc 还支持组合 RowKey 的方式来建表，具体如下：
```
def cat =
  s"""{
     |"table":{"namespace":"default", "name":"iteblog", "tableCoder":"PrimitiveType"},
     |"rowkey":"key1:key2",
     |"columns":{
     |"col00":{"cf":"rowkey", "col":"key1", "type":"string", "length":"6"},
     |"col01":{"cf":"rowkey", "col":"key2", "type":"int"},
     |"col1":{"cf":"cf1", "col":"col1", "type":"boolean"},
     |"col2":{"cf":"cf2", "col":"col2", "type":"double"},
     |"col3":{"cf":"cf3", "col":"col3", "type":"float"},
     |"col4":{"cf":"cf4", "col":"col4", "type":"int"},
     |"col5":{"cf":"cf5", "col":"col5", "type":"bigint"},
     |"col6":{"cf":"cf6", "col":"col6", "type":"smallint"},
     |"col7":{"cf":"cf7", "col":"col7", "type":"string"},
     |"col8":{"cf":"cf8", "col":"col8", "type":"tinyint"}
     |}
     |}""".stripMargin
```
上面的 col00 和 col01 两列组合成一个 rowkey，并且 col00 排在前面，col01 排在后面。比如 col00 ='row002'，col01 = 2，那么组合的 rowkey 为 row002\x00\x00\x00\x02。那么在组合 Rowkey 的查询 shc 都有哪些优化呢？现在我们有如下查询
```
df.sqlContext.sql("select col00, col01, col1 from iteblog where col00 = 'row000' and col01 = 0").show()
```
根据上面的信息，RowKey 其实是由 col00 和 col01 组合而成的，那么上面的查询其实可以将 col00 和 col01 进行拼接，然后组合成一个 RowKey，然后上面的查询其实可以转换成一个 Get 查询。但是在 shc 里面，上面的查询是转换成一个 scan 和一个 get 查询的。scan 的 startRow 为 row000，endRow 为 row000\xff\xff\xff\xff；get 的 rowkey 为 row000\xff\xff\xff\xff，然后再将所有符合条件的数据返回，最后再在 Spark 层面上做一次过滤，得到最后查询的结果。因为 shc 里面组合键查询的代码还没完善，所以当前实现应该不是最终的。

在 shc 里面下面两条 SQL 查询下沉到 HBase 的逻辑一致
```
df.sqlContext.sql("select col00, col01, col1 from iteblog where col00 = 'row000'").show()
df.sqlContext.sql("select col00, col01, col1 from iteblog where col00 = 'row000' and col01 = 0").show()
```
唯一区别是在 Spark 层面上的过滤。

scan 查询优化
如果我们的查询有 < 或 > 等查询过滤条件，比如下面的查询条件：
```
df.sqlContext.sql("select col00, col01, col1 from iteblog where col00 > 'row000' and col00 < 'row005'").show()
```
这个在 shc 里面转换成 HBase 的过滤为一条 get 和 一个 scan，具体为 get 的 Rowkey 为 row0005\xff\xff\xff\xff；scan 的 startRow 为 row000，endRow 为 row005\xff\xff\xff\xff，然后将查询的结果返回到 spark 层面上进行过滤。

总体来说，shc 能在一定程度上对查询进行优化，避免了全表扫描。但是经过评测，shc 其实还有很多地方不够完善，算子下沉并没有下沉到 HBase 层面上进行。目前这个项目正在和 hbase 自带的 connectors 进行整合（https://github.com/apache/hbase-connectors），相关 issue 参见 Enhance the current spark-hbase connector。
