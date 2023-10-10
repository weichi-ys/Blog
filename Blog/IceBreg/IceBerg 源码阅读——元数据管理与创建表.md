> 目前数据湖是一个比较火的方向，它确实能解决传统数仓的很多问题，但相关资料还比较少，尤其是 Iceberg。Iceberg 原理介绍的文章网上已经比较多了，但对于实现细节依然是一头雾水，我准备写一些列关于Iceberg 源码分析的文档，在有了充分的掌握后，再总结分析 Iceberg 是如何解决传统数仓遇到的问题。如遇到任何问题，欢迎在公众号 大数据修炼手册 上与我联系。

在开始正式的分析前，我们先看一下 Iceberg 整体结构图。 metadata 层是 Iceberg 规范的核心，也相对比较复杂一些。

![[Picture/06d784a4d5153cef80adf81aefb9f4d9_MD5.jpg]]

从下往上来看几个核心概念

1.  Data File: 以 Parquet/OCR/AVRO 等格式存储的实际数据内容
2.  Manifest file： 指向多个数据文件，存储了每个数据文件的分区，统计信息等，这些信息主要用来做查询剪裁。
3.  Manifest list：指向多个 Manifest file，主要用来加速 metadata 操作，避免使用文件系统的 list 等
4.  Snapshot: table在某次提交后的一个快照，每一个 snapshot 由一个 Manifest list 文件组成，metadata.json 文件存储了历史所有 snapshot 信息
5.  Catalog：用来做表的元数据查找与持久化

再回头看 Iceberg 的结构图，它其实表示的是在 append 模式下 snapshot 的变化。

1.  第一次写入的时候，可能使用了两个 Exector 进行写入，所以生成了两个 Manifest file 文件，snapshot s0 通过一个 Manifest list 文件指向这两个 Manifest file 文件，间接指向当前状态下的所有数据文件。
2.  第二次执行了追加写，又生成了一批新的数据文件和一个 Manifest file 文件，此时 snapshot s1 通过Manifest list 指向当前 Manifest file 文件和第一次生成的两个 Manifest file 文件

由于涉及到元数据，生产环境一般使用的是 HiveCatalog, 我们把示例代码中的 catalog 切换到 hive。

```
public class IcebergDemo {

    public static void main(String[] args) {
        SparkSession spark = SparkSession
                .builder()
                .master("local")
                .appName("Java Spark SQL basic example")
                .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
                .config("spark.sql.catalog.local", "org.apache.iceberg.spark.SparkCatalog")
                .config("spark.sql.catalog.local.type", "hive")
                .config("spark.sql.catalog.local.uri", "thrift://ip:9083")
                .getOrCreate();

        spark.sql("create database if not exists local.iceberg_db");
        spark.sql("CREATE TABLE local.iceberg_db.table_demo (id bigint, data string) USING iceberg");
        spark.sql("INSERT INTO local.iceberg_db.table VALUES (1, 'a'), (2, 'b'), (3, 'c')");
        Dataset<Row> result = spark.sql("select * from local.iceberg_db.table");
        result.show();

    }
```

同时 pom 依赖中, 加入 metastore 对应版本的 hive 依赖

```
<!-- 记得把parquet  arvo 相关依赖排除掉>
<dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-metastore</artifactId>
    <version>2.1.1</version>
</dependency>
<dependency>
   <groupId>org.apache.hive</groupId>
   <artifactId>hive-exec</artifactId>
   <version>2.1.1</version>
</dependency>
```

## 元数据管理基本概念

### TableMetadata

Iceberg 是一种表规范，所以在 table 是 Iceberg 的核心概念。以 Hive 为例，先思考一下 table 又是由什么组成呢？

很显然一张表由 schema, 分区信息，文件存储路径，文件存储格式，properties 属性，统计信息等。这些信息统称为 metadata。所以 Iceberg 提供 metadata 的抽象，由类`org.apache.iceberg.TableMetadata` 进行表示。先看一个 `TableMetadata` 的一个工厂方法，感受一下 `TableMetadata` 的组成。实际的 `TableMetadata` 包含的信息会更丰富一下，等我们用到的时候再进行分析。

```
//org.apache.iceberg.TableMetadata
static TableMetadata newTableMetadata(Schema schema,
                                        PartitionSpec spec,
                                        SortOrder sortOrder,
                                        String location,
                                        Map<String, String> properties,
                                        int formatVersion)
```

### Table

再说回 Iceberg 的 核心定义 table ，`TableMetadata` 表示的更多的是一些状态，有点类似我们代码中 DTO 的感觉，所以 Iceberg 又抽象了 `org.apache.iceberg.Table` 接口，来表示表的一些操作。`Table` 接口的实现比较多，目前我们只需要关注其中一个实现 BaseTable 即可。下面列出了部分 Table 接口定义的方法，下面展示的主要是一些元数据相关的操作。

![[Picture/a45d904e2086649c0542da4c15ae7bfb_MD5.jpg]]

### TableOperations

其实 BaseTable 把自己的相关操作委托给了 `TableOperations` ，说到底表是有一些元数据组成的，为了查找这些元数据我们需要和 Catalog 进行交互，这些与 Catalog 的交互操作就由 `TableOperations` 来完成，每个 Catalog 都有自己的实现，比如 HiveCatalog 提供的实现就是 `HiveTableOperations` ,我们看一下`TableOperations` 的定义

![[Picture/add5c71e85aa0887cb2d76d7bb4a358b_MD5.jpg]]

核心的方法主要有

1.  current() 通过 Catalog 加载出当前表的 metadata 数据
2.  commit() 数据写入完成后提交当前表，也就是生成一个 snapshot
3.  io() 表示当前表的底层存储介质，比如 HDFS, AWS

### Catalog

Catalog 通常用来保存和查找表的元数据，比如上面提到的 schema,属性信息等。Iceberg 表的元数据主要存储在文件系统上了，Iceberg 的 Catalog 要存储的内容相比 Hive 要轻量很多。 Iceberg 的 catalog 主要有以下作用

1.  metadata 文件地址
2.  表名的存储，可以通过表名获取到表的 metadata 文件地址

从整体上看一下 Iceberg 对 catalog 的抽象

![[Picture/14b94fada7a2ae18fdb46efe772e0e7e_MD5.jpg]]

当引擎层需要用到表的元数据时便会通过 catalog 进行加载，各个引擎都定义了自己的 catalog 规范(接口) ，同时也将 catalog 进行了插件化， Iceberg 为了和引擎层进行对接实现了引擎层定义的接口，如 FlinkCatalog/Spakr catalog 。

`SparkCatalog` 类是 Iceberg 按照 spark 规范实现的用于 spark catalog，主要作用包括

1.  listTable
2.  loadTable
3.  createTable
4.  alertTable
5.  dropTable

Iceberg 为了支持多种 Catalog ,所以也定义了自己的 Catalog 规范, 接口是`org.apache.iceberg.catalog`，定义的方法和 spark 差不多，只是多是事物相关的操作。

关于 catalog 从上往下我们可以这样理解, Iceberg 为了和 Spark 对接,实现了 spark 关于catalog 规范，将这些 catalog 操作委托给了自己的 catalog 实现，目前 Iceberg 支持的 catalog 有 HadooopCatalog/SparkCatalog/JDBCCatalog, 可以通过项来指定。

从catalog 的例子中我们也可以看出 Iceberg 的抽象确实比较优雅干净。

最后我们总结提到的类，以及他们之间的继承关系。TableMetadata 表示一张 Table 的元数据，Table 通过TableOperations 和 Catalog 进行交互，查找和保存这些元数据。

![[Picture/b9caac5e615d55e9f48ce88af992aa7c_MD5.jpg]]

在对元数据概念有了基本了解后，我们以创建表为例 分析一下具体操作

## 创建表

建表语句如下

```
CREATE TABLE local.iceberg_db.table_demo (
id bigint, 
data string
) USING parquet
-- USING 语句用来指定数据文件的格式，支持的选项有 parquet, ocr, avro, iceberg,默认是 parquet
```

SparkSql 从建表SQL语句中解析出表名，表的schema(用`StructType来表示`)，表的属性等信息信息，调用catalog 进行建表。

![[Picture/3e7c0d4a6e3bb8792db3f9b97b055d5f_MD5.jpg]]

在 Iceberg 的 SparkCatalog 中，会用 visitor 模式将 spark 的 schema 转换成 Iceberg 的 schema , 然后通过 TableBuilder 去创建表。这里再挖一个坑 visitor 模式很重要，在 Iceberg 多次用到 ,后面再单独出边博客。

```
//org.apache.iceberg.spark.SparkCatalog 有删减
@Override
  public SparkTable createTable(Identifier ident, StructType schema,
                                Transform[] transforms,
                                Map<String, String> properties) {
    // 将 spark 的schema 装换成 Iceberg 的schema
    Schema icebergSchema = SparkSchemaUtil.convert(schema, useTimestampsWithoutZone);
    Catalog.TableBuilder builder = newBuilder(ident, icebergSchema);
    Table icebergTable = builder
          .withPartitionSpec(Spark3Util.toPartitionSpec(icebergSchema, transforms))
          .withLocation(properties.get("location")) // 显示指定location
          .withProperties(Spark3Util.rebuildCreateProperties(properties)) //指定数据存储的 write.format.default 属性
          .create(); //执行创建的动作
     return new SparkTable(icebergTable, !cacheEnabled);
    } 
  }
```

`BaseMetastoreCatalogTableBuilder` 首先会构建出 TableMetadata 然后交给 TableOperations 向底层catalog 进行提交

```
//BaseMetastoreCatalogTableBuilder 有删减
@Override
    public Table create() {
      TableOperations ops = newTableOps(identifier);
      //如果没有显示指定路径，使用当前数据库的路径
      String baseLocation = location != null ? location : defaultWarehouseLocation(identifier);
      Map<String, String> properties = propertiesBuilder.build();
      //构建 TableMetadata
      TableMetadata metadata = TableMetadata.newTableMetadata(schema, spec, sortOrder, baseLocation, properties);
      ops.commit(null, metadata);

      return new BaseTable(ops, fullTableName(name(), identifier));
    }
```

`HiveTableOperations` commit 操作实际交给了 doCommit 进行执行，主要包含以下操作

1.  把元数据信息写入到存储中，由于是初次建表只会写 metadata.json 文件
2.  将iceberg 的schema 转成Hive metastore表的数据结构
3.  设置表 properties 属性
4.  交给 HiveMetaStoreClient 进行持久化

```
// HiveTableOperations 片段
protected void doCommit(TableMetadata base, TableMetadata metadata) {
  //把元数据信息写入到存储中，由于是初次建表只会写 metadata.json 文件
String newMetadataLocation = writeNewMetadata(metadata, currentVersion() + 1);
  //将iceberg 的schema 转成Hive metastore表的数据结构
tbl.setSd(storageDescriptor(metadata, hiveEngineEnabled));
  // 设置表 properties 属性，重要的是 metadata_location 
setHmsTableParameters(newMetadataLocation, tbl, metadata.properties(), removedProps, hiveEngineEnabled, summary);
//交给 HiveMetaStoreClient 进行持久化
persistTable(tbl, updateHiveTable);
}
```

我们再简单看一下 writeNewMetadata 方法

1.  生成metadata.json 的存储路径
2.  通过 FileIO 创建一个底层存储的文件，FileIO 是iceberg 对底层存储的抽象
3.  将 TableMetadata 对象的数据序列化成 JSON,然后存储在底层

```
//HiveTableOperations
protected String writeNewMetadata(TableMetadata metadata, int newVersion) {
    //生成metadata.json 的存储路径
    String newTableMetadataFilePath = newTableMetadataFilePath(metadata, newVersion);
  //通过 FileIO 创建一个底层存储的文件
    OutputFile newMetadataLocation = io().newOutputFile(newTableMetadataFilePath);
    // 将 TableMetadata 对象的数据序列化成 JSON,然后存储在底层
    TableMetadataParser.overwrite(metadata, newMetadataLocation);

    return newMetadataLocation.location();
  }
```

执行完成后会在hdfs 上生成 /user/hive/warehouse/iceberg\_db.db/iceberg\_demo/metadata/00000-01f71699-40f1-480a-909b-fa0b342c6353.metadata.json

保存的信息主要有 location, schema ,此时 current-snapshot-id 还是空的，因为我们还没有提交过

```
{
  "format-version" : 1,
  "table-uuid" : "d85a326b-9e6e-4291-854b-1e16e82f3ab7",
  "location" : "hdfs://ip:8020/user/hive/warehouse/iceberg_db.db/iceberg_demo",
  "last-updated-ms" : 1641466868200,
  "last-column-id" : 2,
  "schema" : {
    "type" : "struct",
    "schema-id" : 0,
    "fields" : [ {
      "id" : 1,
      "name" : "id",
      "required" : false,
      "type" : "long"
    }, {
      "id" : 2,
      "name" : "data",
      "required" : false,
      "type" : "string"
    } ]
  },
  "current-schema-id" : 0,
  "schemas" : [ {
    "type" : "struct",
    "schema-id" : 0,
    "fields" : [ {
      "id" : 1,
      "name" : "id",
      "required" : false,
      "type" : "long"
    }, {
      "id" : 2,
      "name" : "data",
      "required" : false,
      "type" : "string"
    } ]
  } ],
  "partition-spec" : [ ],
  "default-spec-id" : 0,
  "partition-specs" : [ {
    "spec-id" : 0,
    "fields" : [ ]
  } ],
  "last-partition-id" : 999,
  "default-sort-order-id" : 0,
  "sort-orders" : [ {
    "order-id" : 0,
    "fields" : [ ]
  } ],
  "properties" : {
    "owner" : "app"
  },
  "current-snapshot-id" : -1,
  "snapshots" : [ ],
  "snapshot-log" : [ ],
  "metadata-log" : [ ]
}
```

总结一下创建表的操作流程，iceberg 将spark 传递下来的 schema 信息转换成 TabelMetadata, 然后将 TableMetadata 序列化成 JSON 保存在 底层存储上。

这里发生过3次 schema 转换

1.  第一次是 spark 解析SQL 语句转换成 StructType
2.  Iceberg 将 StructType 转换成自己内部的 schema
3.  Iceberg 将schema ,转换成底层 catalog 的 schema