## 总结
因为SQL语句之间有重复，将相应的执行计划进行缓存，减少重复SQL从“语句 —> 执行计划”的过程时间，提高运行的效率。
适用面：
	重复SQL的情况，例如ETL场景
	常驻服务，有缓存
	
改进：
	存放在远端，进行读取。需要观察解析时长与获取的时间对比

## 背景

[阿里云日志服务（SLS）](https://www.aliyun.com/product/sls)提供一站式数据采集、加工、查询分析、告警、可视化与投递等功能，其中查询分析以简单统一的接口提供大规模数据的查询、计算和分析能力，深受用户喜爱。 目前，分析系统每天接收5+亿次SQL查询请求，在底层，分析系统基于Presto内核，其中Coordinator节点上负载尤其严重，其负责承接用户SQL请求、Query Plan分析、队列管理和任务调度等工作。 ![[Picture/105eda0be790212089a4ef5c912d1c7b_MD5.png]]我们通过对线上实际负载进行Profiling分析，发现Coordinator节点的cpu消耗是其他Worker节点的2-5倍，其中生成Query Plan需要递归遍历整个语法树和Plan节点层级结构，并涉及查询优化和改写，是cpu消耗的主要来源； 另一方面，SLS线上65%的查询请求来自于告警、仪表盘、快速查询等固定SQL，这些是完全一样的SQL请求（周期性调度，仅时间范围不一样）； 因此，我们考虑对SQL进行Query Plan的缓存，以期减少重复SQL的Query Plan计算过程，降低cpu使用率。 本文详细描述了我们在Presto中新增Query Plan Cache的设计和实现以及遇到的问题和解决思路。 

## 什么是Query Plan Cache？

一次SQL执行，一般要经过如下过程：

-   词法/语法解析（parse）-->    生成语法树和语义分析结果
    
-   查询优化/改写（optimize）-->    生成优化/改写后的逻辑执行计划
    
-   生成执行计划（plan）-->    生成最终的物理执行计划
    
-   交由调度器负责执行（execute）
    
-   最终输出结果
    

![[Picture/07e16af19b596e54d7c5d731e099b53e_MD5.png]]其中查询优化/改写是比较耗费cpu和时间的环节，当相同SQL多次请求时，它对应的逻辑/物理执行计划（不同系统实现各异）可能完全一致，我们可以将其缓存起来，以避免多次执行重复工作，来达到减少资源消耗、提升执行速度的效果。

## 方案调研

首先，我们调研了业界的相关方案： **Presto开源社区**目前为止并没有Query Plan Cache的实现，仅有过几次讨论且未产出成熟的方案，参见\[2\],\[3\],\[4\],\[5\]（部分是对Result Cache的讨论）。 

**主流OLAP引擎**如Hive、SparkSQL、Kylin、Impala、Druid等，它们大多数只提供了Query Result Cache，Query Plan Cache这方面的建设一般比较少，这也说得过去，毕竟对于OLAP引擎而言，数据和结果集Cache性价比更高。 

**传统数据库厂商**DB2、Oracle、SQL Server等，这些数据库老炮儿们，在这方面自然有所建树，它们一般都提供了完整的预编译（prepare-statement）、即时查询（ad-hoc SQL）的Plan Cache，有些厂商（如Oracle和SQL Server）甚至还提供了参数绑定（Bind Parameters）的高级功能，但由于是商业软件，大部分设计和实现细节都被隐藏。 MySQL官方貌似没有实现execution plan cahce，但是ADB做了一把实现，参见\[7\]。 PostgreSQL仅提供了prepared statements和plpgsql functions，对于普通SQL并未提供Plan Cache。 

**DB新贵**ClickHouse不提供Plan Cache OceanBase实现了完整的Plan Cache，参见\[8\] TiDB对于预编译和即时查询也提供了Plan Cache，参见\[9\]

## 方案设计

综合上面的方案调研，业界对Plan Cache的典型设计如下： Plan Cache是一个典型的Key-Value结构，Key就是SQL字符串（有的系统提供参数化的能力），Value就是该条 SQL所对应的Plan（不同系统实现中缓存的Plan对象也有所区别，总体上可分为Query Plan和Exection Plan）。 

#### Query Plan vs Execution Plan

二者是有区别的，Query Plan一般指代逻辑执行计划，是根据SQL解析后生成的一个逻辑执行树，一般情况下SQL不变，那这棵树也不会发生变化；而Execution Plan一般指代物理执行计划，其涉及具体的计算任务的分派和执行，一般在大数据系统架构下，往往会根据集群运行时的资源负载、数据节点的分布而动态变化，即使SQL不变，先后两次的执行计划也可能会发生变化。Presto即这样的系统架构，因此我们这里研究对象是Query Plan的缓存。 ![[Picture/bfdc29d7cedaf60efd6d66aa6215edad_MD5.png]]

#### 原生Query Plan生成过程

在Presto中，Query Plan是在Coordinator节点上完成的，一条SQL从开始进入Presto系统到正式被调度执行，其大致分析过程如下图所示： ![[Picture/6d3c658afd21b31bb8280cfb654f2574_MD5.png]]SqlQueryManager，负责接收SQL，创建会话，并做词法分析生成语法树，同时创建出一个SqlQueryExecution实例，交由系统异步执行。 SqlQueryExecution，则真正负责Query的语义分析、Query Plan的生成，最终产生物理执行计划，并交由Coordinator节点负责查询计划的调度和执行。 上图中黄色背景是我们想要Cache住的部分，其中包含词法解析、语义分析、查询优化/改写、计划分段等环节，最终产出的是SQL所对应的逻辑执行计划，即Query Plan。 

#### Query Plan Cache流程设计

我们针对上述分析过程，设计了如下Cache流程： ![[Picture/1ff92ef608675addb2d6028366f3f26a_MD5.png]]1、首先xxhash64计算出SQL文本摘要，作为Plan Cache的Key，并查询Cache 2、若Cache未命中，则走正常的Query Plan过程，并缓存Plan结果（封装成PlanEntry） 3、若Cache命中，则从Cache中获取Query Plan，并进行必要的meta信息的校验（以防止表列信息的更新），若通过则继续进行后续的PalnDistribution，否则缓存失效，并重新执行正常的Query Plan过程 

#### Query Plan Cache数据结构

Plan Cache采用典型的Key-Value结构，Key是SQL文本通过xxhash64生成的Long型数值，Value是如下图中定义的PlanEntry结构，其中

-   planRoot记录一整棵逻辑计划树，它是缓存的主体
    
-   inputs和output记录Query的输入和输出信息，便于过程中需要的时候使用，而无需再遍历整棵树
    
-   metaDigest记录了生成该Query Plan时的meta信息的摘要，用于后续检查Plan Cache是否因Meta信息发生变化而失效。
    

![[Picture/f6989cfa5bf638c74d1346fb5265efdb_MD5.png]]

## 实现细节

#### 缓存隔离级别

Presto设计了三层元数据结构：Catalog->Schema->Table，在SLS的分析服务中，我们通过XXX Connector实现了对应的元数据结构映射，比如用的最多的是SLS Connector，Catalog是SLS，Schema对应用户的ProjectId，Table对应用户Project下的Logstore。 ![[Picture/2b046006400ea4f7a06f3ca9be7d162d_MD5.png]]我们的方案实现了用户Schema下的Query Plan Cache共享，以SLS Connector为例，

-   不同数据源（Connector）相互隔离
    
-   相同数据源（Connector）下不同用户相互隔离
    
-   相同用户下不同Project相互隔离
    
-   相同Project下相同SQL共享Query Plan Cache结果
    

#### 缓存组件和策略选择

我们业务上使用SQL的主要场景如告警、Dashboard、ScheduleSQL等，都具有很强的周期性，一般每5分钟，15分钟，1小时，4小时。调度一次，因此在缓存策略上，我们没有使用采用经典的LRU，而是采用了window-based LFU缓存策略，并自适应窗口，这样可以尽可能地保留长期调度的SQL，以避免短时间内即席查询的干扰。 ![[Picture/9b3a9c1e5e3e10e2af88daa8ce116112_MD5.png]]在实现上，我们并没有打算自己手撸Cache，而是考虑采用主流的JVM Cache组件，他们成熟且已经受过考验。 两个可选的Cache组件：

-   Google cache（缓存策略 LRU）
    
-   Caffeine cache（缓存策略 window-based LFU）
    

我们对缓存组件和策略进行了性能对比，总请求数：7000万，分别在缓存条目为1000、10000和100000的情况下模拟了线上的请求，最终我们采用了caffeine(window-based LFU)作为我们的缓存组件和策略。 ![[Picture/63865aeb3b0f9da0752d1627653fdd1d_MD5.png]]

#### 缓存有效性保证

Query Plan Cache会失效吗？会的！ Presto是一个开放引擎，能支持多种数据源，而元数据信息（如数据分布、表列信息等）都是交由各个数据源自行维护，然后通过SPI集成进来。 那么问题来了：当数据分布或表列信息发生变化后，原先缓存的Query Plan可能就会失效。 例如：正常创建表-->查询-->缓存Query Plan，在此之后修改了表列，将可能导致缓存失效。 为了保证Query Plan缓存的有效性，我们增加了元数据信息检查机制，在首次进行Query Plan分析过程中，会对SQL涉及到的元数据产生信息摘要（包括指纹和lastUpdateTime），并保存在PlanEntry的metaDigest中。后续在使用该Plan Cache前，需要先进行metaDigest校验，如果不一致表示元数据发生过更新，该Plan Cache强制失效并逐出，然后重新进行Query Plan过程并缓存。 

#### 运维管理

我们实现了Query Plan Cache的一整套配套设施： 1、 设置系统级别的plan cache开关（支持系统禁用plan cache功能） 2、设置session级别的plan cache开关（支持客户端禁用plan cache功能） 3、设置plan cache缓存大小和策略（默认size=10000，window-based LFU） 4、支持plan cache的实时指标监控（包括缓存命中率、数量、内存占用大小等指标） 5、提供RESTful API支持删除单条缓存、清空全部缓存条目 

## 问题和挑战

#### Query Plan Cache不是万金油（有限制的支持部分SQL操作）

我们在实现过程中，发现一些SQL操作并不适合做Query Plan Cache，比如： 日期时间相关的函数和表达式（select ... where t < now()...），它们一般会在查询优化/改写过程中就被Constant值所替代，这样相同的SQL可能Query Plan并不相同，一种解法是将日期时间表达式的改写推迟到物理执行阶段，但这样就无法实现查询优化，在我们的系统中还会影响数据分布，从性价比出发，我们决定不支持。 Query Plan Cache有限制的支持部分SQL操作： 1、支持所有SELECT语句（下述部分不支持的函数和表达式除外） 2、不支持日期时间相关的函数和表达式 3、不支持时序补全函数（time\_series） 4、不支持UPDATE，INSERT，DELETE以及DDL（这类SQL在我们的业务中使用极少） 5、不支持Explain、SHOW等管理命令

#### 缓存给JVM GC带来的影响

Plan Cache灰度上线后，通过持续观察，发现Plan Cache对GC会产生压力，经过长达1个多月的持续调查和分析，最终我们找到了原因：Plan Cache常驻JVM内存，逐步会进入JVM老年代，因Plan Cache把很多原先的Query分析过程给拦掉了，young区一直保持很低水位，old区持续上涨，这个过程中，没有其他GC，只有程序每隔20s发起的System.gc()（主要用于回收CodeCache），而这个Explicit GC会清除并发标记，打断JVM mixedGC回收old区的工作，old区死对象无法被回收，导致old区内存持续上涨至顶从而触发FullGC。 ![[Picture/d2ac58066a2ef8312212d4cebdff4f73_MD5.png]]，在内存分析过程给了我们非常细心和详尽的技术支持，Explicit GC打断mixedGC的源码也翻出来了。 针对这个问题，我们做了三个方面的改进： 1、精简Plan Cache缓存对象的内存大小 我们发现以上现象仅在部分集群上出现，而在有些集群上表现正常，因此我们统计对比了两个集群上的Plan Cache对象大小，在问题集群上PlanEntry的平均RetainedSize高达137K，其中tableColumns列信息占用了接近50%的内存空间，还存在一些其他无用的冗余对象，通过MetaDigest我们去除了冗余的内存占用，有效精简了Plan Cache对象的内存大小，从137K精简到30K以下。 ![[Picture/e0d1c359afb8aff6faf99dccc5a0e61e_MD5.png]]2、控制Plan Cache内存占用 1）PlanCache缓存条目数从100K减至10K，通过对比测试，发现HitRate仅降低1%，但却大大减少了内存占用。 2）对Caffeine进行了改造，通过后台线程监控JVM水位，并自适应调整缓存条目数（水位高时减少缓存条目数），这块还特地请教了Caffeine原作者Ben Manes，也感谢他给了非常中肯的意见和指导，参见\[10\] 

3、JVM GC优化 1）将System.gc()的调用间隔拉长（从20s拉长到60s），这样避免了mixedGC被打断，让old区能够得到有效回收 2）SqlQueryExecution做完之后，不一定立马释放，这可能会导致相关联的PlanEntry被延迟GC，从而被动进入old区，因此我们在每次使用完之后主动清空PlanEntry=null，加速PlanEntry的GC 

## 优化效果

目前，Query Plan Cache已经完成全网灰度，线上表现稳定，缓存命中率保持在70%左右 ![[Picture/319157b669de78192f9ddb315da01a25_MD5.png]]具体的优化效果：

<table id="56919c53a6l04"><colgroup colwidth="1*"></colgroup><colgroup colwidth="1*"></colgroup><tbody><tr id="56921182a6v0d"><td id="5691ea71a6iu6" rowspan="1" colspan="1"><p id="5691ea70a6ffk">平均CPU使用率节省20-30%（4-5个core）</p></td><td id="56921181a6lsg" rowspan="1" colspan="1"><p id="56921180a6itc">有效减少Coordinator节点压力</p></td></tr><tr id="56921187a6d7k"><td id="56921184a6kyp" rowspan="1" colspan="1"><p id="56921183a6b9c">平均Query Plan分析延时减少60%</p></td><td id="56921186a6ylc" rowspan="1" colspan="1"><p id="56921185a6ker">平均延时从16ms降低至5ms</p></td></tr><tr id="5692118ca6nhl"><td id="56921189a6wtd" rowspan="1" colspan="1"><p id="56921188a6ilt">平均Query执行延时减少15%</p></td><td id="5692118ba6cm5" rowspan="1" colspan="1"><p id="5692118aa6swu">平均延时从320ms降低至270ms</p></td></tr></tbody></table>

![[Picture/779f6181f75b89686271b8e47263c576_MD5.png]]

![[Picture/60d60da796be7236a4c41eee3685e0a3_MD5.png]]

## 参考

\[1\] 阿里云日志服务（SLS）：[https://www.aliyun.com/product/sls](https://www.aliyun.com/product/sls)

\[2\] [Presto query cache discussion](https://gist.github.com/luohao/62f02599d8bc72de07054c402fcf18aa)

\[3\] [Presto for low latency: How to do plan caching or parameter substitution in presto ?](https://github.com/trinodb/trino/issues/1141)

\[4\] [Generate query digests to enable caching query results](https://github.com/prestodb/presto/pull/2645)

\[5\] [Caching in Presto](https://www.qubole.com/blog/caching-presto/)

\[6\] Planning For Reuse：[https://use-the-index-luke.com/blog/2011-07-16/planning-for-reuse](https://use-the-index-luke.com/blog/2011-07-16/planning-for-reuse)

\[7\] MySQL · 特性分析 · 执行计划缓存设计与实现：[http://mysql.taobao.org/monthly/2016/09/04/](http://mysql.taobao.org/monthly/2016/09/04/)

\[8\] OceanBase：[https://github.com/oceanbase/oceanbase/tree/master/src/sql/plan\_cache](https://github.com/oceanbase/oceanbase/tree/master/src/sql/plan_cache)

\[9\] [TiDB SQL Prepare Execution Plan Cache](https://docs.pingcap.com/tidb/stable/sql-prepare-plan-cache)

\[10\] [Does caffeine can dynamicly adjustment cache size by JVM memory watermark?](https://github.com/ben-manes/caffeine/discussions/564)

\[11\] [Understanding SQL Server query plan cache](https://www.sqlshack.com/understanding-sql-server-query-plan-cache/)

\[12\] [SQL Server Plan Caching and Recompilation--Plan Cache](https://logicalread.com/sql-server-plan-cache-w01/)

\[13\] [SQL Server Plan Caching and Recompilation–Parameterization](https://logicalread.com/sql-server-plan-caching-parameterization-w01/)

\[14\] [SQL Query Optimization Techniques in SQL Server: Parameter Sniffing](https://www.sqlshack.com/query-optimization-techniques-in-sql-server-parameter-sniffing/)