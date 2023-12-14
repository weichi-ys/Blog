## 一、概述

HBase是基于列式存储的分布式数据库，底层存储采用的是LSM树，是Hadoop生态下核心技术之一。

### 1.1 架构图

![[Blog/Picture/71aed4ed4ec844a2e55cc1bb12a3793a_MD5.jpg]]

### 1.2 组件介绍

HBase由三种类型的服务器以主从模式构成：

-   Region Server：负责数据的读写服务，用户通过与Region server交互来实现对数据的访问。
-   HBase HMaster：负责Region的分配及数据库的创建和删除等操作。
-   ZooKeeper：负责维护集群的状态（某台服务器是否在线，服务器之间数据的同步操作及master的选举等）。

HDFS的DataNode负责存储所有Region Server所管理的数据，即HBase中的所有数据都是以HDFS文件的形式存储的。出于使Region server所管理的数据更加本地化的考虑，Region server是根据DataNode分布的。HBase的数据在写入的时候都存储在本地。但当某一个region被移除或被重新分配的时候，就可能产生数据不在本地的情况。这种情况只有在所谓的compaction之后才能解决。

## 二、HMaster

HMaster负责region的分配，数据库的创建和删除操作，其职责包括：

-   调控Region server的工作

-   在集群启动的时候分配region，根据恢复服务或者负载均衡的需要重新分配region。
-   监控集群中的Region server的工作状态。（通过监听zookeeper对于ephemeral node状态的通知）。

-   管理数据库

-   提供创建，删除或者更新表格的接口。

如下图所示：

![[Blog/Picture/68bb1faaa063b00acea0fe5b4164d6fe_MD5.jpg]]

## 三、Zookeeper

HBase利用ZooKeeper维护集群中服务器的状态并协调分布式系统的工作，其功能如下：

-   维护服务器是否存活，是否可访问的状态并提供服务器故障/宕机的通知。
-   使用一致性算法来保证服务器之间的同步。
-   负责Master选举的工作。

需要注意的是要保证良好的一致性及顺利的Master选举，集群中的服务器数目必须是奇数（如三台或五台）。

如下图所示：

![[Blog/Picture/5ef12a726acadbbd6859820f235430a8_MD5.jpg]]

### 3.1 组件间的协作

ZooKeeper用来协调分布式系统的成员之间共享的状态信息。Region Server及HMaster也与ZooKeeper连接。ZooKeeper通过心跳信息为活跃的连接维持相应的ephemeral node。如下图所示：

![[Blog/Picture/a31ab9c930ede57abe4d94bc3074c5c3_MD5.jpg]]

每一个Region server都在ZooKeeper中创建相应的ephemeral node。HMaster通过监控这些ephemeral node的状态来发现正常工作的或发生故障下线的Region server。

HMaster之间通过互相竞争创建ephemeral node进行Master选举。ZooKeeper会选出区中第一个创建成功的作为唯一一个活跃的HMaster。活跃的HMaster向ZooKeeper发送心跳信息来表明自己在线的状态。不活跃的HMaster则监听活跃HMaster的状态，并在活跃HMaster发生故障下线之后重新选举，从而实现了HBase的高可用性。

如果Region server或者HMaster不能成功向ZooKeeper发送心跳信息，则其与ZooKeeper的连接超时之后与之相应的ephemeral node就会被删除。监听ZooKeeper状态的其他节点就会得到相应node不存在的信息，从而进行相应的处理。活跃的HMaster监听Region Server的信息，并在其下线后重新分配Region server来恢复相应的服务。不活跃的HMaster监听活跃HMaster的信息，并在起下线后重新选出活跃的HMaster进行服务。

## 四、Region Server

HBase中的表是根据row key的值水平分割成所谓的region的。一个region包含表中所有row key位于region的起始键值和结束键值之间的行。

集群中负责管理Region的结点叫做Region server。Region server负责数据的读写。每一个Region server大约可以管理1000个region。Region的结构如下图所示：

![[Blog/Picture/60a5f72ee10387e53f1c600fa2250020_MD5.jpg]]

### 4.1 组成部分

Region server由如下几个部分组成：

-   WAL：既Write Ahead Log。WAL是HDFS分布式文件系统中的一个文件，即HLog。WAL用来存储尚未写入永久性存储区中的新数据。WAL也用来在服务器发生故障时进行数据恢复。
-   Block Cache：Block cache是读缓存。Block cache将经常被读的数据存储在内存中来提高读取数据的效率。当Block cache的空间被占满后，其中被读取频率最低的数据将会被杀出。
-   MemStore：MemStore是写缓存。其中存储了从WAL中写入但尚未写入硬盘的数据。MemStore中的数据在写入硬盘之前会先进行排序操作。每一个region中的每一个column family对应一个MemStore。
-   Hfiles：Hfiles存在于硬盘上，根据排序号的键存储数据行。

如下图所示：

![[Blog/Picture/05674bc181444ba30dfead1a5dfe7884_MD5.jpg]]

### 4.2 数据读写

### 4.2.1 META Table

HBase中有一个特殊的起目录作用的表格，称为META table。META table中保存集群region的地址信息。ZooKeeper中会保存META table的位置。

META table中保存了HBase中所有region的信息，格式类似于B tree。其结构如下：

-   键：region的起始键，region id。
-   值：Region server

如下图所示：

![[Blog/Picture/36371d99b0931164844775a6245f8ca5_MD5.jpg]]

### 4.2.2 第一次读写

当用户第一次从HBase中进行读或写操作时，执行以下步骤：

1.  客户端从ZooKeeper中得到保存META table的Region server的信息。
2.  客户端向该Region server查询负责管理自己想要访问的row key的所在的region的Region server的地址。客户端会缓存这一信息以及META table所在位置的信息。
3.  客户端与负责其row所在region的Region Server通信，实现对该行的读写操作。

在未来的读写操作中，客户端会根据缓存寻找相应的Region server地址。除非该Region server不再可达。这时客户端会重新访问META table并更新缓存。这一过程如下图所示：

![[Blog/Picture/2060f84ad55a0098f5ca3344fcd0139a_MD5.jpg]]

### 4.2.3 写操作

**步骤一**

当HBase的用户发出一个 PUT 请求时（也就是HBase的写请求），HBase进行处理的第一步是将数据写入HBase的write-ahead log（WAL）中。

-   WAL文件是顺序写入的，也就是所有新添加的数据都被加入WAL文件的末尾。WAL文件存在硬盘上。
-   当server出现问题之后，WAL可以被用来恢复尚未写入HBase中的数据（因为WAL是保存在硬盘上的）。

如下图所示：

![[Blog/Picture/4ca41e275389a3932bcc1d6bfc9fabba_MD5.jpg]]

  
**步骤二**

当数据被成功写入WAL后，HBase将数据存入MemStore。这时HBase就会通知用户PUT操作已经成功了。

过程如下图所示：

![[Blog/Picture/23a812e6d4930956dc74ca156bd49bc4_MD5.jpg]]

### 4.2.4 读合并（Read Merge）和读放大（Read amplification）

HBase中对应于某一行数据的cell可能位于多个不同的文件或存储介质中。比如已经存入硬盘的行位于硬盘上的HFile中，新加入或更新的数据位于内存中的MemStore中，最近读取过的数据则位于内存中的Block cache中。所以当我们读取某一行的时候，为了返回相应的行数据，HBase需要根据Block cache，MemStore以及硬盘上的HFile中的数据进行所谓的读合并操作。

1.  HBase会首先从Block cache（HBase的读缓存）中寻找所需的数据。
2.  接下来，HBase会从MemStore中寻找数据。因为作为HBase的写缓存，MemStore中包含了最新版本的数据。

如果HBase从Block cache和MemStore中没有找到行所对应的cell所有的数据，系统会接着根据索引和 bloom filter（布隆过滤器） 从相应的HFile中读取目标行的cell的数据。

如下图所示：

![[Blog/Picture/e2e0c89ccd848a7e87d58b79f711a0fe_MD5.jpg]]

这里一个需要注意读放大效应（Read amplification）。根据前文所说，一个MemStore对应的数据可能存储于多个不同的HFile中（由于多次的flush），因此在进行读操作的时候，HBase可能需要读取多个HFile来获取想要的数据。这会影响HBase的性能表现。

如下图所示：

![[Blog/Picture/2544ba1329ef87fe32bfd4810faa904e_MD5.jpg]]

### 4.4 MemStore flush到磁盘

MemStore存在于内存中，其中存储的是按键排好序的待写入硬盘的数据。数据也是按键排好序写入HFile中的。每一个Region中的每一个Column family对应一个MemStore文件。因此对数据的更新也是对应于每一个Column family。

如下图所示：

![[Blog/Picture/a686fc623219add0076fe5956123e2b2_MD5.jpg]]

当MemStore中积累了足够多的数据之后，整个MemCache中的数据会被一次性写入到HDFS里的一个新的HFile中。因此HDFS中一个Column family可能对应多个HFile。这个HFile中包含了相应的cell，或者说键值的实例。这些文件随着MemStore中积累的对数据的操作被flush到硬盘上而创建。

需要注意的是，MemStore存储在内存中，这也是为什么HBase中Column family的数目有限制的原因。每一个Column family对应一个MemStore，当MemStore存满之后，里面所积累的数据就会一次性flush到硬盘上。同时，为了使HDFS能够知道当前哪些数据已经被存储了，MemStore中还保存最后一次写操作的序号。

每个HFile中最大的序号作为meta field存储在其中，这个序号标明了之前的数据向硬盘存储的终止点和接下来继续存储的开始点。当一个region启动的时候，它会读取每一个HFile中的序号来得知当前region中最新的操作序号是什么（最大的序号）。

如下图所示：

![[Blog/Picture/de365cfdaa524fb9db5b46931e29462f_MD5.jpg]]

### 4.5 HFile

上面已经说过，当MemStore中积累足够多的数据的时候就会将其中的数据整个写入到HDFS中的一个新的HFile中。因为MemStore中的数据已经按照键排好序，所以这是一个顺序写的过程。由于顺序写操作避免了磁盘大量寻址的过程，所以这一操作非常高效。

如下图所示：

![[Blog/Picture/1135a1b80c127d5e4084863dc5a42721_MD5.jpg]]

注：有些文章会提到StoreFile。StoreFile是以HFile格式保存在HDFS上，实际上是对HFile做了轻量级封装。

### 4.5.1 HFile的结构

HFile中包含了一个多层索引系统。这个多层索引是的HBase可以在不读取整个文件的情况下查找数据。这一多层索引类似于一个B+树。

-   键值对根据键大小升序排列。
-   索引指向64KB大小的数据块。
-   每一个数据块还有其相应的叶索引（leaf-index）。
-   每一个数据块的最后一个键作为中间索引（intermediate index）。
-   根索引（root index）指向中间索引。

文件结尾指向meta block。因为meta block是在数据写入硬盘操作的结尾写入该文件中的。文件的结尾同时还包含一些别的信息。比如 bloom filter 及时间信息。bloom filter可以帮助HBase加速数据查询的速度。因为HBase可以利用 bloom filter 跳过不包含当前查询的键的文件。时间信息则可以帮助HBase在查询时跳过读操作所期望的时间区域之外的文件。

如下图所示：

![[Blog/Picture/4b5d6faef3fe9bfdc378862c9f5e2ea1_MD5.jpg]]

### 4.5.2 HFile的索引

HFile的索引在HFile被打开时会被读取到内存中。这样就可以保证数据检索只需一次硬盘查询操作。

如下图所示：

![[Blog/Picture/6ee933bfdebe1e7610e20699d3d93604_MD5.jpg]]

### 4.4 Compaction

### 4.4.1 Minor Compaction

HBase会自动选取一些较小的HFile进行合并，并将结果写入几个较大的HFile中。这一过程称为Minor compaction。Minor compaction通过Merge sort的形式将较小的文件合并为较大的文件，从而减少了存储的HFile的数量，提升HBase的性能。这一过程如下图所示：

![[Blog/Picture/27c1436063b79691ae2e3f4cc2750ef6_MD5.jpg]]

4.4.2 Major Compaction

所谓Major Compaction指的是HBase将对应于某一个Column family的所有HFile重新整理并合并为一个HFile，并在这一过程中删除已经删除或过期的cell，更新现有cell的值。这一操作大大提升读的效率。但是因为Major compaction需要重新整理所有的HFile并写入一个HFile，这一过程包含大量的硬盘I/O操作以及网络数据通信。这一过程也称为写放大（Write amplification）。在Major compaction进行的过程中，当前Region基本是处于不可访问的状态。

Major compaction可以配置在规定的时间自动运行。为避免影响业务，Major compaction一般安排在夜间或周末进行。

需要注意的一点是，Major compaction会将当前Region所服务的所有远程数据下载到本地Region server上。这些远程数据可能由于服务器故障或者负载均衡等原因而存储在于远端服务器上。

这一过程如下图所示：

![[Blog/Picture/71be643ec2e2f03db2da27aad97c72c9_MD5.jpg]]

### 4.5 Region的分割

回顾下Region：

-   HBase中的表格可以根据行键水平分割为一个或几个region。每个region中包含了一段处于某一起始键值和终止键值之间的连续的行键。
-   每一个region的默认大小为1GB。
-   相应的Region server负责向客户端提供访问某一region中的数据的服务。
-   每一个Region server能够管理大约1000个region（这些region可能来自同一个表格，也可能来自不同的表格）。

如下图所示：

![[Blog/Picture/5fcac65c6a816ffc583db2eb8d53af5f_MD5.jpg]]

每一个表格最初都对应于一个region。随着region中数据量的增加，region会被分割成两个子region。每一个子region中存储原来一半的数据。同时Region server会通知HMaster这一分割。出于负载均衡的原因，HMaster可能会将新产生的region分配给其他的Region server管理（这也就导致了Region server服务远端数据的情况的产生）。

如下图所示：

![[Blog/Picture/961e5abeee1b42f75c834c3f35426280_MD5.jpg]]

## 五、事务

HBase目前只支持行级事务，强一致性，满足的ACID特性。其采用了WAL（Write Ahead Log）策略，以及通过锁和MVCC机制来实现并发控制。

### 5.1 事务原子性的保证

HBase数据会首先写入WAL，再写入MemStore。写入MemStore出现异常时，很容易回滚，因此保证写入/更新原子性，只需要保证写入WAL的原子性即可。

当前版本WAL结构（之前的版本有区别，这里不讨论）：

```xml
<logseq#-for-entire-txn>:<WALEdit-for-entire-txn> <logseq#-for-entire-txn>:<-1, 3, <Keyvalue-for-edit-c1>, <KeyValue-for-edit-c2>, <KeyValue-for-edit-c3>>
```

每个事务只会产生一个WAL单元，以此来保证WAL写入的原子性。

WAL持久化策略：

-   **SKIP\_WAL**：表示不写WAL，这样写入更新性能最好，但在RegionServer宕机的时候有可能会丢失部分数据；
-   **ASYNC\_WAL**：表示异步将WAL持久化到硬盘，因为是异步操作所以在异常的情况下也有可能丢失少量数据；
-   **SYNC\_WAL**：表示同步将WAL持久化到操作系统缓存，再由操作系统将数据异步持久化到磁盘，这种场景下RS宕掉并不会丢失数据，当操作系统宕掉会导致部分数据丢失；
-   **FSYNC\_WAL**：表示WAL写入之后立马落盘，性能相对最差。目前实现中FSYNC\_WAL并没有实现（是说HBase会丢数据？而且不就一行代码的实现了吗？）！用户可以根据业务对数据丢失的敏感性在客户端配置相应的持久化策略。

### 5.2 写写并发控制

多个写入/更新同时进行会导致数据不一致的问题，HBase通过获取行锁来实现写写并发，如果获取不到，就需要不断重试等待或者自旋等待，直至其他线程释放该锁。拿到锁之后开始写入数据，写入完成之后释放行锁即可。这种行锁机制是实现写写并发控制最常用的手段，MySQL也使用了行锁来实现写写并发。

HBase支持批量写入/更新，实现批量写入的并发控制也是使用行锁。但这里需要注意的是必须使用两阶段锁协议，步骤如下：

(1) 获取所有待写入/更新行记录的行锁。  
(2) 开始执行写入/更新操作。  
(3) 写入完成之后再统一释放所有行记录的行锁。

不能更新一行锁定（释放）一行，多个事务之间容易形成死锁。两阶段锁协议就是为了避免死锁，MySQL事务写写并发控制同样使用两阶段锁协议。

### 5.3 读写并发控制

读写之间也需要一种并发控制来保证读取的数据总能够保持一致性，读写并发不能通过加锁的方式，性能太差，采用的是MVCC机制（Mutil Version Concurrent Control）。HBase中MVCC机制实现主要分为两步：

(1) 为每一个写入/更新事务分配一个Region级别自增的序列号。  
(2) 为每一个读请求分配一个已完成的最大写事务序列号。

如下图所示：

![[Blog/Picture/b5217192e0ba8d039b77b0dbc8548c3d_MD5.jpg]]

  
图中两个写事务分别分配了序列号1和序列号2，读请求进来的时候事务1已经完成，事务2还未完成，因此分配事务1对应的序列号1给读请求。此时序列号1对本次读可见，序列号2对本次读不可见。

所有的事务都会生成一个Region级别的自增序列，并添加到队列中，如下图最左侧队列，其中最底端为已经提交的事务，队列中的事务为未提交事务。现假设当前事务编号为15，并且写入完成（中间队列红色框框），但之前的写入事务还未完成（序列号为12、13、14的事务还未完成），此时当前事务必须等待，而且对读并不可见，直至之前所有事务完成之后才会对读可见（即读请求才能读取到该事务写入的数据）。如最右侧图，15号事务之前的所有事务都成功完成，此时Read Point就会移动到15号事务处，表示15号事务之前的所有改动都可见。  

![[Blog/Picture/81cb265509eedc8dfb4e06c33b15e96a_MD5.jpg]]

## 六、备份与恢复

### 6.1 数据备份（Data Replication）

HDFS中所有的数据读写操作都是针对主节点进行的。HDFS会自动备份WAL和HFile。HBase基于HDFS来提供可靠的安全的数据存储。当数据被写入HDFS本地时，另外两份备份数据会分别存储在另外两台服务器上。

如下图所示：

![[Blog/Picture/8b35e89eb76afce72282a5f7e6b3b3b0_MD5.jpg]]

### 6.2 异常恢复（Crash Recovery）

WAL文件和HFile都存储于硬盘上且存在备份，因此恢复它们是非常容易的。那么HBase如何恢复位于内存中的MemStore呢？

![[Blog/Picture/e1b0b5666789e7d78cf8d91cd6f7aa38_MD5.jpg]]

当Region Server宕机的时候，其所管理的region在这一故障被发现并修复之前是不可访问的。ZooKeeper负责根据服务器的心跳信息来监控服务器的工作状态。当某一服务器下线之后，ZooKeeper会发送该服务器下线的通知。HMaster收到这一通知之后会进行恢复操作。  
HMaster会首先将宕机的Region Server所管理的region分配给其他仍在工作的活跃的Region Server。然后HMaster会将该服务器的WAL分割并分别分配给相应的新分配的Region Server进行存储。新的Region Server会读取并顺序执行WAL中的数据操作，从而重新创建相应的MemStore。

如下图所示：

![[Blog/Picture/36a012baff23d546d271d2c25cbf4b67_MD5.jpg]]

### 6.3 数据恢复（Data Recovery）

WAL文件之中存储了一系列数据操作。每一个操作对应WAL中的一行。新的操作会顺序写在WAL文件的末尾。

那么当MemStore中存储的数据因为某种原因丢失之后应该如何恢复呢？HBase以来WAL对其进行恢复。相应的Region server会顺序读取WAL并执行其中的操作。这些数据被存入内存中当前的MemStore并排序。最终当MemStore存满之后，这些数据被flush到硬盘上。

如下图所示：

![[Blog/Picture/b008d247107fe16c909a6b8481d65c0c_MD5.jpg]]

**优点**

-   强一致性模型

-   当一个写操作得到确认时，所有的用户都将读到同一个值。

-   可靠的自动扩展

-   当region中的数据太多时会自动分割。
-   使用HDFS分布存储并备份数据。

-   内置的恢复功能

-   使用WAL进行数据恢复。

-   与Hadoop集成良好

-   MapReduce在HBase上非常直观。

**缺点**

-   WAL恢复较慢。
-   异常恢复复杂且低效。
-   需要进行占用大量资源和大量I/O操作的Major compaction。

## 八、参考

-   <u><a href="https://link.zhihu.com/?target=https%3A//mapr.com/blog/in-depth-look-hbase-architecture/" target="_blank" rel="nofollow noreferrer">深入理解HBase的系统架构</a></u>（原文）
-   <u><a href="https://link.zhihu.com/?target=https%3A//blog.csdn.net/Yaokai_AssultMaster/article/details/72877127" target="_blank" rel="nofollow noreferrer">深入理解HBase的系统架构</a></u>（翻译）
-   <u><a href="https://link.zhihu.com/?target=http%3A//www.itboth.com/d/F7rAFb/hbase" target="_blank" rel="nofollow noreferrer">HBase架构核心模块</a></u>
-   <u><a href="https://link.zhihu.com/?target=https%3A//blogs.apache.org/hbase/entry/apache_hbase_internals_locking_and" target="_blank" rel="nofollow noreferrer">Apache HBase Internals: Locking and Multiversion Concurrency Control</a></u>
-   <u><a href="https://link.zhihu.com/?target=http%3A//hbasefly.com/2017/07/26/transaction-2/" target="_blank" rel="nofollow noreferrer">数据库事务系列－HBase行级事务模型</a></u>