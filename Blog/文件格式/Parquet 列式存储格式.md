Parquet 是 Hadoop 生态圈中主流的列式存储格式，最早是由 Twitter 和 Cloudera 合作开发，2015 年 5 月从 Apache 孵化器里毕业成为 Apache 顶级项目。

有这样一句话流传：如果说 HDFS 是[大数据](https://cloud.tencent.com/solution/bigdata?from_column=20065&from=20065)时代文件系统的事实标准，Parquet 就是大数据时代存储格式的事实标准。

## 01

## **整体介绍**

先简单介绍下：

-   Parquet 是一种支持嵌套结构的列式存储格式
-   非常适用于 OLAP 场景，按列存储和扫描

诸如 Parquet 这种列存的特点或优势主要体现在两方面。

**1、更高的压缩比**

列存使得更容易对每个列使用高效的压缩和编码，降低磁盘空间。（网上的case是不压缩、gzip、snappy分别能达到11/27/19的压缩比）

**2、更小的IO操作**

使用映射下推和谓词下推，只读取需要的列，跳过不满足条件的列，能够减少不必要的数据扫描，带来性能的提升并在表字段比较多的时候更加明显。

> 关于映射下推与谓词下推： 映射下推，这是列式存储最突出的优势，是指在获取数据时只需要扫描需要的列，不用全部扫描。 谓词下推，是指通过将一些过滤条件尽可能的在最底层执行以减少结果集。谓词就是指这些过滤条件，即返回bool：true和false的表达式，比如SQL中的大于小于等于、Like、Is Null等。

## 02

## **项目概述**

Parquet 是与语言无关的，而且不与任何一种数据处理框架绑定在一起，适配多种语言和组件，能够与 Parquet 适配的查询引擎包括 Hive, Impala, Pig, Presto, Drill, Tajo, HAWQ, IBM Big SQL等，计算框架包括 MapReduce, Spark, Cascading, Crunch, Scalding, Kite 等，数据模型包括 Avro, Thrift, Protocol Buffer, POJOs 等。

Parquet 的项目组成及自下而上交互的方式如图所示：

![[Picture/835fa153807bb3b29849f4c44e2d3cfd_MD5.png]]

这里可以将其分为三层。

-   [数据存储](https://cloud.tencent.com/product/cdcs?from_column=20065&from=20065)层：定义 Parquet 文件格式，其中元数据在 parquet-format 项目中定义，包括 Parquet 原始类型定义、Page类型、编码类型、压缩类型等等。
-   对象转换层：这一层在 parquet-mr 项目中，包含多个模块，作用是完成其他对象模型与 Parquet 内部数据模型的映射和转换，Parquet 的编码方式使用的是 striping and assembly 算法。
-   对象模型层：定义如何读取 Parquet 文件的内容，这一层转换包括 Avro、Thrift、Protocal Buffer 等对象模型/序列化格式、Hive serde 等的适配。并且为了帮助大家理解和使用，Parquet 提供了 org.apache.parquet.example 包实现了 java 对象和 Parquet 文件的转换。

其中，对象模型可以简单理解为内存中的数据表示，Avro, Thrift, Protocol Buffer, Pig Tuple, Hive SerDe 等这些都是对象模型。例如 parquet-mr 项目里的 parquet-pig 项目就是负责把内存中的 Pig Tuple 序列化并按列存储成 Parquet 格式，以及反过来把 Parquet 文件的数据反序列化成 Pig Tuple。

这里需要注意的是 Avro, Thrift, Protocol Buffer 等都有他们自己的存储格式，但是 Parquet 并没有使用他们，而是使用了自己在 parquet-format 项目里定义的存储格式。所以如果你的项目使用了 Avro 等对象模型，这些数据序列化到磁盘还是使用的 parquet-mr 定义的转换器把他们转换成 Parquet 自己的存储格式。

## 03

## **支持嵌套的数据模型**

Parquet 支持嵌套结构的数据模型，而非扁平式的数据模型，这是 Parquet 相对其他列存比如 ORC 的一大特点或优势。支持嵌套式结构，意味着 Parquet 能够很好的将诸如 Protobuf，thrift，json 等对象模型进行列式存储。

Parquet 的数据模型也是 schema 表达方式，用关键字 message 表示。每个字段包含三个属性，repetition属性（required/repeated/optional）、数据类型（primitive基本类型/group复杂类型）及字段名。

```
message AddressBook {
 required string owner;
 repeated string ownerPhoneNumbers;
 repeated group contacts {
   required string name;
   optional string phoneNumber;
 }
}
```

这个 schema 中每条记录表示一个人的 AddressBook。有且只有一个 owner，owner 可以有 0 个或者多个 ownerPhoneNumbers，owner 可以有 0 个或者多个 contacts。每个 contact 有且只有一个 name，这个 contact 的 phoneNumber 可有可无。这个 schema 可以用下面的树结构来表示。

![[Picture/541f65794b44ff5e02ebf9d775109c6b_MD5.png]]

Parquet 格式的数据类型没有复杂的 Map, List, Set 等，而是使用 repeated fields 和 groups 来表示。例如 List 和 Set 可以被表示成一个 repeated field，Map 可以表示成一个包含有 key-value 对的 repeated field，而且 key 是 required 的。

## 04

## **存储模型**

这里存储模型又可以理解为存储格式或文件格式，Parquet 的存储模型主要由行组（Row Group）、列块（Column Chuck）、页（Page）组成。

1、行组，Row Group：Parquet 在水平方向上将数据划分为行组，默认行组大小与 HDFS Block 块大小对齐，Parquet 保证一个行组会被一个 Mapper 处理。

2、列块，Column Chunk：行组中每一列保存在一个列块中，一个列块具有相同的数据类型，不同的列块可以使用不同的压缩。

```
3、页，Page：Parquet 是页存储方式，每一个列块包含多个页，一个页是最小的编码的单位，同一列块的不同页可以使用不同的编码方式。
```

另外 Parquet 文件还包含header与footer信息，分别存储文件的校验码与Schema等信息。参考官网的一张图：

![[Picture/08ef0f8bdf1a426c4b6d9fb6984d8e77_MD5.png]]

关于 Parquet 的存储模型暂且了解到这个程度，更深入的细节可参考文末的链接。

## 05

## **Parquet vs ORC**

除了 Parquet，另一个常见的列式存储格式是 ORC（OptimizedRC File）。在 ORC 之前，Apache Hive 中就有一种列式存储格式称为 RCFile（RecordColumnar File），ORC 是对 RCFile 格式的改进，主要在压缩编码、查询性能方面做了优化。因此 ORC/RC 都源于 Hive，主要用来提高 Hive 查询速度和降低 Hadoop 的数据存储空间。

Parquet 与 ORC 的不同点总结以下：

-   嵌套结构支持：Parquet 能够很完美的支持嵌套式结构，而在这一点上 ORC 支持的并不好，表达起来复杂且性能和空间都损耗较大。
-   更新与 ACID 支持：ORC 格式支持 update 操作与 ACID，而 Parquet 并不支持。
-   压缩与查询性能：在压缩空间与查询性能方面，Parquet 与 ORC 总体上相差不大。可能 ORC 要稍好于 Parquet。
-   查询引擎支持：这方面 Parquet 可能更有优势，支持 Hive、Impala、Presto 等各种查询引擎，而 ORC 与 Hive 接触的比较紧密，而与 Impala 适配的并不好。之前我们说 Impala 不支持 ORC，直到 [CDH](https://cloud.tencent.com/product/cdh?from_column=20065&from=20065) 6.1.x 版本也就是 Impala3.x 才开始以 experimental feature 支持 ORC 格式。

关于 Parquet 与 ORC，首先建议根据实际情况进行选择。另外，根据笔者的综合评估，如果不是一定要使用 ORC 的特性，还是建议选择 Parquet。

## 06

最后介绍下社区的一个 Parquet 开源工具，主要用于查看 Parquet 文件元数据、Schema 等。

使用方法：

```
#Runfrom Hadoop
hadoop jar ./parquet-tools-<VERSION>.jar --help
hadoop jar ./parquet-tools-<VERSION>.jar <command> my_parquet_file.parq
#Runlocally
java -jar ./parquet-tools-<VERSION>.jar --help
java -jar ./parquet-tools-<VERSION>.jar <command> my_parquet_file.parq
```

比如：

```
$ hadoop jar parquet-tools-1.8.0.jar schema 20200515160701.parquet               
message t_staff_info_partition {

  optional int64 age;
  optional binary dt (UTF8);
  optional int64 id;
  optional binary name (UTF8);
  optional binary updated_time (UTF8);
}
```

jar包地址可访问 [https://www.mvnjar.com/org.apache.parquet/parquet-tools/jar.html](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fwww.mvnjar.com%2Forg.apache.parquet%2Fparquet-tools%2Fjar.html)。

## 参考文档

[https://parquet.apache.org/documentation/latest](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fparquet.apache.org%2Fdocumentation%2Flatest)

[https://blog.twitter.com/2013/dremel-made-simple-with-parquet](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fblog.twitter.com%2F2013%2Fdremel-made-simple-with-parquet)

[https://www.infoq.cn/article/in-depth-analysis-of-parquet-column-storage-format/](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fwww.infoq.cn%2Farticle%2Fin-depth-analysis-of-parquet-column-storage-format%2F)

[https://docs.cloudera.com/documentation/enterprise/latest/topics/impala\_file\_formats.html](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fdocs.cloudera.com%2Fdocumentation%2Fenterprise%2Flatest%2Ftopics%2Fimpala%255C_file%255C_formats.html)

![[Picture/e9aebef125fd04661207afd81f395102_MD5.png]]

本文参与 [腾讯云自媒体分享计划](https://cloud.tencent.com/developer/support-plan)，分享自微信公众号。
