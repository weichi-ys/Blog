 系列专题：[数据湖系列文章](https://blog.csdn.net/younger_china/article/details/125956749 "数据湖系列文章")

___

        在使用不同的引擎进行大数据计算时，需要将数据根据计算引擎进行适配。这是一个相当棘手的问题，为此出现了一种新的解决方案：**介于上层计算引擎和底层存储格式之间的一个中间层**。这个中间层不是数据存储的方式，只是定义了数据的元数据组织方式，并向计算引擎提供统一的类似传统数据库中"表"的语义。它的底层仍然是[Parquet](https://so.csdn.net/so/search?q=Parquet&spm=1001.2101.3001.7020)、ORC等存储格式。

基于此，Netflix开发了Iceberg，目前已经是Apache的顶级项目，  
        https://iceberg.apache.org/

## 1\. Iceberg是什么

> **Apache Iceberg is an open table format for huge analytic datasets.** Iceberg adds tables to compute engines including Flink, Trino, Spark and Hive using a high-performance table format that works just like a SQL table.

        Iceberg是一种开放的[数据湖](https://so.csdn.net/so/search?q=%E6%95%B0%E6%8D%AE%E6%B9%96&spm=1001.2101.3001.7020)表格式。可以简单理解为是基于计算层（Flink , Spark）和存储层（ORC，Parqurt，Avro）的一个中间层，用Flink或者Spark将数据写入Iceberg，然后再通过其他方式来读取这个表，比如Spark，Flink，Presto等。

![[Picture/e6e80db7a40d0555617a0ba02984b103_MD5.png]]

 在文件Format（parquet/avro/[orc](https://so.csdn.net/so/search?q=orc&spm=1001.2101.3001.7020)等）之上实现Table语义：

1.  支持定义和变更Schema
2.  支持Hidden Partition和Partition变更

1.  ACID语义
2.  历史版本回溯

1.  借助partition和columns统计信息实现分区裁剪
2.  不绑定任何存储引擎，可拓展到HDFS/S3/OSS等
3.  容许多个writer并发写入，乐观锁机制解决冲突。

## 2. Iceberg的Table Format介绍

        Iceberg是为分析海量数据而设计的，被定义为Table Format，Table Format介于计算层和存储层之间。

        Table Format向下管理在存储系统上的文件，向上为计算层提供丰富的接口。存储系统上的文件存储都会采用一定的组织形式，譬如读一张Hive表的时候，HDFS文件系统会带一些Partition，数据存储格式、数据压缩格式、数据存储HDFS目录的信息等，这些信息都存在Metastore上，Metastore就可以称之为一种文件组织格式。

        一个优秀的文件组织格式，如Iceberg，可以更高效的支持上层的计算层访问磁盘上的文件，做一些list、rename或者查找等操作。

        表和表格式是两个概念。表是一个具象的概念，应用层面的概念，我们天天说的表是简单的行和列的组合。而表格式是数据库系统实现层面一个抽象的概念，它定义了一个表的Scheme定义：包含哪些字段，表下面文件的组织形式（Partition方式）、元数据信息（表相关的统计信息，表索引信息以及表的读写API），如下图左侧所示：

![[Picture/b744afbcb65782afaa26119163235156_MD5.png]]

         上图右侧是Iceberg在数据仓库生态中的位置，和它差不多相当的一个组件是Metastore。不过Metastore是一个服务，而Iceberg就是一系列jar包。对于Table Format，可以认为主要包含4个层面的含义，分别是表schema定义（是否支持复杂数据类型），表中文件的组织形式，表相关统计信息、表索引信息以及表的读写API信息。

-   表schema定义了一个表支持字段类型，比如int、string、long以及复杂数据类型等。
-   表中文件组织形式最典型的是Partition模式，是Range Partition还是Hash Partition。

-   Metadata数据统计信息。
-   表的读写API。上层引擎通过对应的API读取或者写入表中的数据。

## 3. Iceberg的核心思想

        Iceberg的核心思想，就是在时间轴上跟踪表的所有变化：

-   快照表示表数据文件的一个完整集合。
-   每次更新操作会生成一个新的快照。

## 4\. Iceberg的元数据管理

        从图中可以看到Iceberg将数据进行分层管理，主要分为元数据管理层和数据存储层。元数据管理层又可以细分为三层：

-   Metadata File
    
-   Snapshot
    

-   Manifest
    

        Metadata File存储当前版本的元数据信息（所有snapshot信息）；Snapshot表示当前操作的一个快照，每次commit都会生成一个快照，一个快照中包含多个Manifest。每个Manifest中记录了当前操作生成数据所对应的文件地址，也就是data files的地址。基于snapshot的管理方式，Iceberg能够进行time travel（历史版本读取以及增量读取），并且提供了serializable isolation。  
数据存储层支持不同的文件格式，目前支持Parquet、ORC、AVRO。

## 5\. Iceberg的重要特性

        Apache Iceberg设计初衷是**为了解决Hive离线数仓计算慢的问题**，经过多年迭代已经发展成为构建数据湖服务的表格式标准。关于Apache Iceberg的更多介绍，请参见Apache Iceberg官网。

目前Iceberg提供以下核心能力：

## 5.1 丰富的计算引擎

-   优秀的内核抽象使之不绑定特定引擎，目前在支持的有Spark、Flink、Presto、Hive；
-   Iceberg提供了Java native API，不用特定引擎也可以访问Iceberg表。

## 5.2 灵活的文件组织形式

-   提供了基于流式的增量计算模型和基于批处理的全量表计算模型，批任务和流任务可以使用相同的存储模型（HDFS、OZONE——OZone 是 Hadoop 社区重点投入开发的下一代存储引擎），数据不再孤立，以构建低成本的轻量级数据湖存储服务；
-   Iceberg支持隐藏分区（Hidden Partitioning）和分区布局变更（Partition Evolution），方便业务进行数据分区策略更新；

-   支持Parquet、ORC、Avro等存储格式。

## 5.3 优化数据入湖流程

-   Iceberg提供ACID事务能力，上游数据写入即可见，不影响当前数据处理任务，这大大简化了ETL；
-   Iceberg提供upsert/merge into行级别数据变更，可以极大地缩小数据入库延迟。

## 5.4 增量读取处理能力

-   Iceberg支持通过流式方式读取增量数据，实现主流开源计算引擎入湖和分析场景的完善对接；
-   Spark struct streaming支持；

-   Flink table source支持；

-   支持历史版本回溯。

## 6. 数据文件结构

        先了解一下Iceberg在文件系统中的布局，总体来讲Iceberg分为两部分数据，第一部分是数据文件，如下图中的 parquet 文件。第二部分是表元数据文件（Metadata 文件），包含 Snapshot 文件（snap-\*.avro）、Manifest 文件(\*.avro)、TableMetadata 文件(\*.json)等。

![[Picture/036226ffba1039e163746b7afbfa37a5_MD5.png]]

##  6.1 元数据文件

        其中metadata目录存放元数据管理层的数据，表的元数据是不可修改的，并且始终向前迭代；当前的快照可以回退。

### 6.1.1 Table Metadata

        version\[number\].metadata.json：存储每个版本的数据更改项。

### 6.1.2 快照（SnapShot）

        snap-\[snapshotID\]-\[attemptID\]-\[commitUUID\].avro：存储快照snapshot文件;

        快照代表一张Iceberg表在某一时刻的状态。也被称为清单列表（Manifest List），里面存储的是清单文件列表，每个清单文件占用一行数据。清单列表文件以snap开头，以avro后缀结尾，每次更新都产生一个清单列表文件。每行中存储了清单文件的路径。

        清单文件里面存储数据文件的分区范围、增加了几个数据文件、删除了几个数据文件等信息。数据文件（Data Files）存储在不同的Manifest Files里面，Manifest Files存储在一个Manifest List文件里面，而一个Manifest List文件代表一个快照。

### 6.1.3 清单文件（Manifest File）

        \[commitUUID\]-\[attemptID\]-\[manifestCount\].avro：manifest文件

        清单文件是以avro格式进行存储的，以avro后缀结尾，每次更新操作都会产生多个清单文件。其里面列出了组成某个快照（snapshot）的数据文件列表。每行都是每个数据文件的详细描述，包括数据文件的状态、文件路径、分区信息、列级别的统计信息（比如每列的最大最小值、空值数等）、文件的大小以及文件里面数据的行数等信息。其中列级别的统计信息在 Scan 的时候可以为算子下推提供数据，以便可以过滤掉不必要的文件。

## 6.2 数据文件

        data目录组织形式类似于hive，都是以分区进行目录组织（图中dt为分区列）

        Iceberg的数据文件通常存放在data目录下。一共有三种存储格式（Avro、Orc和Parquet），主要是看您选择哪种存储格式，后缀分别对应avro、orc或者parquet。在一个目录，通常会产生多个数据文件。

## 7\. 参考文献

\[01\]  [https://www.toutiao.com/article/7099724190609539591](https://www.toutiao.com/article/7099724190609539591 "https://www.toutiao.com/article/7099724190609539591")