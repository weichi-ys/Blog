**MVCC**

就是多版本并发控制。MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问。

为什么需要MVCC呢？数据库通常使用锁来实现隔离性。最原生的锁，锁住一个资源后会禁止其他任何线程访问同一个资源。但是很多应用的一个特点都是读多写少的场景，很多数据的读取次数远大于修改的次数，而读取数据间互相排斥显得不是很必要。所以就使用了一种读写锁的方法，读锁和读锁之间不互斥，而写锁和写锁、读锁都互斥。这样就很大提升了系统的并发能力。之后人们发现并发读还是不够，又提出了能不能让读写之间也不冲突的方法，就是读取数据时通过一种类似快照的方式将数据保存下来，这样读锁就和写锁不冲突了，不同的事务session会看到自己特定版本的数据。当然快照是一种概念模型，不同的数据库可能用不同的方式来实现这种功能。

**InnoDB与MVCC**

MVCC只在 READ COMMITTED 和 REPEATABLE READ 两个隔离级别下工作。其他两个隔离级别够和MVCC不兼容, 因为 READ UNCOMMITTED 总是读取最新的数据行, 而不是符合当前事务版本的数据行。而 SERIALIZABLE 则会对所有读取的行都加锁。

**Redo log, bin log, Undo log**

InnoDB中通过undo log实现了数据的多版本，而并发控制通过锁来实现。

undo log除了实现MVCC外，还用于事务的回滚。MySQL Innodb中存在多种日志，除了错误日志、查询日志外，还有很多和数据持久性、一致性有关的日志。

binlog，是mysql服务层产生的日志，常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的, 另外通过解析binlog能够实现mysql到其他数据源（如ElasticSearch)的数据复制。

redo log记录了数据操作在物理层面的修改，mysql中使用了大量缓存，缓存存在于内存中，修改操作时会直接修改内存，而不是立刻修改磁盘，当内存和磁盘的数据不一致时，称内存中的数据为脏页(dirty page)。为了保证数据的安全性，事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中, redo log是按照顺序写入的，磁盘的顺序读写的速度远大于随机读写。当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。这样实现了事务的原子性、一致性和持久性。

undo log: 除了记录redo log外，当进行数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC。

**版本链与Undo log**

innodb中通过B+树作为索引的数据结构，并且主键所在的索引为ClusterIndex(聚簇索引), ClusterIndex中的叶子节点中保存了对应的数据内容。一个表只能有一个主键，所以只能有一个聚簇索引，如果表没有定义主键，则选择第一个非NULL唯一索引作为聚簇索引，如果还没有则生成一个隐藏id列作为聚簇索引。

除了Cluster Index外的索引是Secondary Index(辅助索引)。辅助索引中的叶子节点保存的是聚簇索引的叶子节点的值。

InnoDB行记录中除了刚才提到的rowid外，还有trx\_id和db\_roll\_ptr, trx\_id表示最近修改的事务的id,db\_roll\_ptr指向undo segment中的undo log。

新增一个事务时事务id会增加，trx\_id能够表示事务开始的先后顺序。

Undo log分为Insert和Update两种，delete可以看做是一种特殊的update，即在记录上修改删除标记。

update undo log记录了数据之前的数据信息，通过这些信息可以还原到之前版本的状态。

当进行插入操作时，生成的Insert undo log在事务提交后即可删除，因为其他事务不需要这个undo log。

进行删除修改操作时，会生成对应的undo log，并将当前数据记录中的db\_roll\_ptr指向新的undo log。

![[Picture/Pasted image 20230925114944.png]]

**ReadView**

说完了undo log我们再来看看ReadView。已提交读和可重复读的区别就在于它们生成ReadView的策略不同。

ReadView中主要就是有个列表来存储我们系统中当前活跃着的读写事务，也就是begin了还未提交的事务。通过这个列表来判断记录的某个版本是否对当前事务可见。其中最主要的与可见性相关的属性如下：

**up\_limit\_id**：当前已经提交的事务号 + 1，事务号 < up\_limit\_id ，对于当前Read View都是可见的。理解起来就是创建Read View视图的时候，之前已经提交的事务对于该事务肯定是可见的。

**low\_limit\_id**：当前最大的事务号 + 1，事务号 >= low\_limit\_id，对于当前Read View都是不可见的。理解起来就是在创建Read View视图之后创建的事务对于该事务肯定是不可见的。

**trx\_ids**：为活跃事务id列表，即Read View初始化时当前未提交的事务列表。所以当进行RR读的时候，trx\_ids中的事务对于本事务是不可见的（除了自身事务，自身事务对于表的修改对于自己当然是可见的）。理解起来就是创建RV时，将当前活跃事务ID记录下来，后续即使他们提交对于本事务也是不可见的。

用一张图更好的理解一下：

![[Picture/Pasted image 20230925114957.png]]

最后我们来举个例子让我们更好理解上面的内容。

比如我们有如下表：

![[Picture/Pasted image 20230925115014.png]]

现在有一个事务id是60的执行如下语句并提交：

```
update user set name = '强哥1' where id = 1;
```

此时undo log存在版本链如下：

![[Pasted image 20230925115027.png]]

提交事务id是60的记录后，接着有一个事务id为100的事务，修改name=强哥2，但是事务还没提交。则此时的版本链是：

![[Pasted image 20230925115038.png]]

此时另一个事务发起select语句查询id=1的记录，因为trx\_ids当前只有事务id为100的，所以该条记录不可见，继续查询下一条，发现trx\_id=60的事务号小于up\_limit\_id，则可见，直接返回结果强哥1。

那这时候我们把事务id为100的事务提交了，并且新建了一个事务id为110也修改id为1的记录name=强哥3，并且不提交事务。这时候版本链就是：

![[Pasted image 20230925115057.png]]

这时候之前那个select事务又执行了一次查询,要查询id为1的记录。

如果你是已提交读隔离级别READ\_COMMITED，这时候你会重新一个ReadView，那你的活动事务列表中的值就变了，变成了\[110\]。按照上的说法，你去版本链通过trx\_id对比查找到合适的结果就是强哥2。

如果你是可重复读隔离级别REPEATABLE\_READ，这时候你的ReadView还是第一次select时候生成的ReadView,也就是列表的值还是\[100\]。所以select的结果是强哥1。所以第二次select结果和第一次一样，所以叫可重复读！

也就是说已提交读隔离级别下的事务在每次查询的开始都会生成一个独立的ReadView,而可重复读隔离级别则在第一次读的时候生成一个ReadView，之后的读都复用之前的ReadView。

这就是Mysql的MVCC,通过版本链，实现多版本，可并发读-写，写-读。通过ReadView生成策略的不同实现不同的隔离级别。

参考：

[人类身份验证 - SegmentFault](https://link.zhihu.com/?target=https%3A//segmentfault.com/a/1190000012650596)

[面试官：谈谈你对Mysql的MVCC的理解？](https://link.zhihu.com/?target=https%3A//baijiahao.baidu.com/s%3Fid%3D1629409989970483292%26wfr%3Dspider%26for%3Dpc)

[MySQL MVCC机制 · Issue #68 · zhangyachen/zhangyachen.github.io](https://link.zhihu.com/?target=https%3A//github.com/zhangyachen/zhangyachen.github.io/issues/68)

[何登成的技术博客 " InnoDB多版本(MVCC)实现简要分析](https://link.zhihu.com/?target=http%3A//hedengcheng.com/%3Fp%3D148%23_Toc322691905)
