![[Picture/d9b7497bd7af5a7cc0adbbf773d42960_MD5.jpg]]

Iceberg 官方对 Iceberg 的定义是

> Iceberg is a high-performance format for huge analytic tables.

我个人对 Iceberg 的理解:

Iceberg 是一种表格式的规范，以及实现了这种规范的代码库，通过提供了一组 API 供计算引擎或其它进程调用。Iceberg 通过元数据文件给数据文件加了一层索引。

这里提到表格式，那什么是表格式呢？让我们从 Hive 讲起

## **Hive 表以及存在的问题**

### Hive 如何定义一张表

在 Iceberg 之前 Hive 表是大数据领域通用的表格式，让我们看看 Hive 如何定义一张表？整体上可以分为两部分：

1.  存放在 metastore 中的 schema ，属性信息，文件路径等。
2.  保存在文件系统上指定目录下的数据文件。

![[Picture/b972106642abacd1cd9478e8a836d089_MD5.jpg]]

### **Hive 表存在的问题**

1.  现在的计算引擎(Presto Spark)都是分布式执行的，以 Spark 为例，假如某个表有 100 个数据文件，执行时一共有 10 个 Executor，在 Exector 执行前，Driver 会对数据文件进行切分，最终每个 Executor 可能分配 10 个数据文件。由于 Hive 表格式只保存了数据文件的目录，所以在文件切分前会使用文件系统的 list 操作，列出所有的数据文件。

![[Picture/6894792696c4e9a4794b293e04ff1742_MD5.jpg]]

list 操作存在以下问题：

-   HDFS 文件系统下，如果频繁大量的调用 list 操作会给 NameNode 的 RPC 带来压力；如果文件数过多，list 会比较耗时，曾经就遇到一个这样的例子，由于一个表使用了二级分区，生成了大量的分区和文件，在执行全表扫描的时候，仅仅是文件切分就花了10 分钟。
-   对象存储下 list 操作非常慢

2\. 由于 Hive 表格式只保存了数据文件的目录，所以在 Executor 执行时，先把计算结果写入临时目录，等待 Executor 全部执行完成后，Driver 端会把临时文件目录 rename 到正式的文件目录，此操作依赖文件系统的 rename 操作。在对象存储中 rename 操作非常慢。

小结一下，由于 list 和 rename 在对象存储上的性能问题，基本上无法直接使用成本更低的对象存储来替代 HDFS 存储。

3\. Hive 表的 schema 集中存储在 metastore 中，metastore 很容易成为性能瓶颈，同时也会带来分库分表等运维成本。

4 Hive 表的统计信息(文件行数，文件大小，文件个数) 不是强制要求写入的，很多情况不存在统计信息或者是过时的，planner 层无法有效的做基于代价的优化。另外统计信息的粒度很粗是表级别的。

5\. 不支持删除，更新表等操作，或者成本非常高，需要重刷数据

6\. 如果同时存在 读-写，写-写 任务时，无法保证任务的一致性，会发生莫名其妙的错误，或刚写入的数据被其它任务覆盖了。

上面我们分析了 Hive 如何定义一张表以及存在的问题，下面我们看看 Iceberg 是如何定义一张表。

以下面的 schema 为例进行讲解，首先使用 spark 创建一张表 ，再使用 spark 执行两次插入操作。

```
CREATE TABLE local.iceberg_db.events (
event_time timestamp,
device_id bigint)
USING ICEBERG
PARTITIONED BY (
  days(event_time),
  bucket(64,device_id))

INSERT INTO local.iceberg_db.events VALUES (current_timestamp(), 1), (current_timestamp(), 2), (current_timestamp(), 3)
INSERT INTO local.iceberg_db.events VALUES (current_timestamp(), 1), (current_timestamp(), 2), (current_timestamp(), 3)
```

### **文件组织结构**

在 Insert 执行完成后，最终生成的文件结构，如下图所示，主要可以分为三类文件:

1.  数据文件，普通的 Parquet 文件，存放着写入的数据。
2.  元数据文件，主要是 avro 和 json 类型，这正是 Iceberg 表和 Hive 表的本质区别。
3.  catalog（version-hint.txt 文件，只有使用 Hadoop catalog 才会存在此文件）

Iceberg 文件组织结构图

![[Picture/9dcb781be9ebf7c10795e903147c4f97_MD5.jpg]]

### **Iceberg 整体架构**

上面的元数据文件之间是什么关系，是如何组织的呢？我们结合的 Iceberg 的架构图从底向上进行讲解。

Iceberg 整体架构图

![[Picture/8c03bee6f35ae277b3dd51fb61d6d698_MD5.jpg]]

**manifest file**

首先是 manifest file，Iceberg 文件组织结构图中16db143c,18ce4c4a 开头的 avro 文件。manifest file 文件核心内容如下图所示，

一个 manifest file 可以索引多个数据文件，在 manifest file 文件中一行索引一个数据文件，例如第一行，表示 01-data.parquet 所属的分区，每一列的最大值和最小值。

还有一些其它信息，如文件大小，文件行数，null 值行数等信息，这里没有做详细的展示。默认一个 manifest file 文件大小是 8M。

![[Picture/b0c40c043dcb19cab72f505aa2d0d634_MD5.jpg]]

**manifestlist file**

Iceberg 文件组织结构图中以 snap- 开头的 avro 文件。 manifestlist file 的内容 和 manifest file 文件很相似，不同的是 manifestlist file 是对 manifest file 文件的索引，**一个快照只能有一个 manifestlist file 文件。**

![[Picture/37803c4ab639ec9fb0f50a5763750ffa_MD5.jpg]]

**metadata file**

Iceberg 文件组织结构图中 的 json 文件，每次提交表的时候生成一个，上面一共有 3 个 metadata.json 文件，创建表的时候生成一个，每 insert 一次生成一个。包含的信息主要有：

1.  当前表的 schema 信息和历史的 schema 信息。
2.  表的提交历史（快照）和当前快照ID。快照中包含当前表的一些统计信息，例如文件总大小，文件个数，总行数。
3.  文件路径、分区等其它信息。

![[Picture/f97e5f2303b79523352e0f17dda0bd5b_MD5.jpg]]

**catalog**

Iceberg 文件组织结构图中 version-hint.txt 文件，使用 Hadoop Catalog 才会生成此文件。由于表的 schema 和统计信息已经保存在 metadata.josn 文件中， catalog 比较轻量，只需要保存 Iceberg 表最新的 metadata.json 文件的存储路径。

Iceberg 当前支持的catalog：

1.  Hive Metastore
2.  Hadoop (只要提供原子的 rename 操作的文件系统都可以)
3.  JDBC

上面讲解了 Iceberg 表的元数据文件组织结构，基于上面的组织结构可以给 Iceberg 代码哪些特性呢？

## **Iceberg 的特性**

### 1\. 快照（snapshot）

再回到 Iceberg 整体架构图中，在 metadata file 中画了 S0 S1 两个符号，S 就是 snapshot 的首字母表示快照。那什么是快照呢？

快照表示一张 Iceberg 表当前状态的 schema 以及持有的数据文件。

在 Iceberg 的架构图片中，可以认为执行了两次 append 写入，快照 S0 表示持有最左边的那堆数据文件，快照 S1 表示持有下面所有的数据文件。

如果是 overwrite 写入，快照 S0 和 S1 会持有完全不同的数据文件，即使是 Overwrite 写入，Iceberg 也不会在写入时清理之前的文件，快照会一直保留着，用户可以很方便的回滚到之前的快照。快照的清理需要管理者自己启动任务进行清理的，比如可以保留最近的 10 个快照。

### 2\. 上云（对象存储

计算引擎在做任务切分时不再依赖 list 操作，直接读取 manifest 文件即可拿到所有的数据文件。

数据文件提交时不再需要 rename 操作，而是通过生成快照的方式，就是生成相应的元数据文件。

解决了这两个问题为使用成本更低的对象存储打下了基础。

### 3 开放

Iceberg 的 catalog 比较轻量，只需要存储一个文件路径，可以不再依赖中心化的 Hive Metastore。

shema 信息和数据文件是存放在一起，实现数据自治。

### 4 统计信息与二级索引

在生成 Iceberg 元数据时，统计信息如 记录数, 文件大小等信息是强制写入的，所以总是实时的，对计算引擎代价优化非常友好。

拥有数据文件级别的二级索引，目前有 Min-Max 索引，在执行引擎计算时可以协助做文件级别的过滤，更多索引正在开发中。

### 5 增量读取

我们以下图为例，通过比较 snapshot1 和 snapshot2 可以发现增量写入的文件，这也是 Iceberg 一个很大的特性，通过比较 snapshot ，可以发现两个版本之间数据文件的差异，比如找出新增和删除的文件。增量读取具有非常广泛的应用场景，例如结合 Flink 构建实时数仓。 但在 Hive 中只能以微分区的方式。

![[Picture/e7372cffb3daea5662ccefd2747dfe92_MD5.jpg]]

### 6 SCHEMA 变更

支持 schema 变更，不需要全部重刷数据。

以下面的场景为例，一开始我们的数据量比较小，分区是按照月进行的，后来数据量增大后我们希望改成天分区，如果是 Hive 表的话，我们只能重刷整个数据。Iceberg 只需要执行一条 alter 语句，老数据保持不变，新写入的数据按照天进行分区。查询的时候老分区还是按月进行。

![[Picture/99d301b1b7c6830853a785149f6a3f30_MD5.jpg]]

### **7\. 隐藏分区**

在 Hive 的查询中如果需要使用分区过滤，必须在 SQL 中指明分区，需要使用者对数据 schema 有深入了解。在下面的 Hive 表的 schema 中，分区字段 event\_time\_day\_par 实际上是从 event\_time 派生出来的，但在第一个查询中无法使用到分区过滤，必须像第二个查询一样必须带上 分区字段。

```
CREATE TABLE iceberg_db.events (
event_time timestamp,
device_id bigint)
PARTITIONED BY (event_time_day_par string)

select * from iceberg_db.events where event_time > '2022-03-08'
select * from iceberg_db.events where event_time > '2022-03-08' and event_time_day_par > '2022-03-08'
```

Iceberg 的解决方案，Iceberg 提供了一些分区函数，在下面的例子中，我们可以看到分区字段是从 event\_time 字段中派生出来的，在查询时可以通过 event\_time 字段自动推导出所属的分区。

```
CREATE TABLE local.iceberg_db.events (
event_time timestamp,
device_id bigint)
USING ICEBERG
PARTITIONED BY (days(event_time))

select * from iceberg_db.events where event_time > '2022-03-08'
```

### **8 Time Travel**

Iceberg 保留历史的快照，可以很方便的在快照之间穿梭

```
spark.read
    .option("snapshot-id", 10963874102873L)
    .format("iceberg")
    .load("path/to/table")
```

### **9\. 事物 ACID**

由于有快照机制提供了串行化的隔离级别，

读-读：无影响

读-写：每次读取都读取到当前快照下的文件，读写相互隔离，写入不会影响正在进行的读

写-写：基于同一个快照的写，只有一个能提交成功

![[Picture/d288f17b8fdce3e224a30227aaa2f782_MD5.jpg]]

### 10\. 更新(COPY ON WRITE)

```
delete from local.iceberg_db.iceberg_demo where id = 1
```

1.  Iceberg 根据过滤条件和数据文件的统计信息，过滤出只包含过滤条件的数据文件，假如只包含数据文件f1
2.  在过滤出的数据文件 f1 上，对过滤条件取反进行查询，将查询结果写入新的文件
3.  将步骤一中过滤出的数据文件标记为删除，将步骤二中新写入的文件和之前不包含过滤条件的数据文件 merge 后作为新的数据文件

![[Picture/687a60e6afbb8f1bc9cad35c779dd894_MD5.jpg]]

### **11\. 更新（MERGE ON READ）**

merge on read 实现比较复杂，感兴趣的t同学可以查看讨论文档。

[https://docs.google.com/document/d/1FMKh\_SQ6xSUUmoCA8LerTkzIxDUN5JbStQp5Hzot4eo/edit#heading=h.p74qmh3a6ets](https://link.zhihu.com/?target=https%3A//docs.google.com/document/d/1FMKh_SQ6xSUUmoCA8LerTkzIxDUN5JbStQp5Hzot4eo/edit%23heading%3Dh.p74qmh3a6ets)

## 总结

我们从 Hive 表格式讲起，讲解了 Hive 表格式存在的问题，Iceberg 表格式是如何解决的，以及 Iceberg 表格式带来的特性。我们总结一写 Iceberg 表格式：

**_Iceberg 通过元数据文件给数据文件加了一层索引。_**

## 参考内容

[Metadata Indexing in Iceberg](https://link.zhihu.com/?target=https%3A//tabular.io/blog/iceberg-metadata-indexing/)