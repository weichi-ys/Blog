[真正让你明白Hive调优系列3：笛卡尔乘积,小表join大表，Mapjoin等问题](https://my.oschina.net/u/4373790/blog/4335307 "真正让你明白Hive调优系列3：笛卡尔乘积,小表join大表，Mapjoin等问题")

### 0.Hive中的优化分类

     **真正想要掌握Hive的优化，要熟悉相关的MapReduce，Yarn，hdfs底层源码，明晰Hive的底层执行流程。真正让你明白Hive调优系列，会征对下面分类逐一分析演示。**

#### 大类1：参数优化

1.  **文件输入前看是否需要map前合并小文件**
2.  **控制map个数，根据实际需求确认每个map的数据处理量，split的参数等**
3.  **Map输出是否需要启动压缩，减少网络传输，OOM处理等**
4.  **控制redcue个数，控制每个reduce的吞吐量，OOM处理等**
5.  **是否将common-join转换成map-join处理策略**
6.  **文件输出是否需要启动小文件合并策略**
7.  **其他相关参数的配置：如严格模式，JVM重用，列剪切等**

#### 大类2：开发中优化

1.  **数据倾斜，这个是Hive优化的重头戏。出现的原因是因为出现了数据的重新分发和分布，启动了redcue。Hive中数据倾斜分类：group by ，count(distinct)以及join产生的数据倾斜（当然一些窗口函数中用了partition by一会造成数据倾斜）**
2.  j**oin相关的优化**：分类大表join大表，小表join大表的优化
3.  **代码细节优化分类**：**比如去重用group by替代distinct ；**多表关联，先进行子查询后再进行关联**；表关联时一定要在子查询里过滤掉NULL值，避免数据倾斜；**不要对一个表进行重复处理，多使用临时表，尽量做到一次处理多次使用等等，

#### 1.笛卡尔乘积与小表join大表

 Hive 设定为严格模式（hive.mapred.mode=strict）时，不允许在 HQL 语句中出现笛卡尔积， 这实际说明了 Hive 对笛卡尔积支持较弱。因为找不到 Join key，Hive 只能使用 1 个 reducer 来完成笛卡尔积。

**需求1**：**一个小表join大表**，且两个表特殊的是笛卡尔乘积（on true/on 1=1）。小表的数据量2Mb，大表的数据是4Gb左右。实际开发中该段代码跑了3个小时左右

```
drop table if exists FDM_TMP.TMP_FSA_MULTI_PATH_FUNL_ANALYSE_RSLT_D_21_${hivevar:statis_date};CREATE TABLE IF NOT EXISTS FDM_TMP.TMP_FSA_MULTI_PATH_FUNL_ANALYSE_RSLT_D_21_${hivevar:statis_date}stored as rcfile asSELECT T1.ACCT_NO                                                     ,T1.PAGE_ID                                                        ,T1.PAGE_NAME                                                      ,T1.PAGE_URL                        ,T1.TRMNL_TYPE                                                     ,T1.DEV_ID        ,T0.PATH_ID                                                      ,T0.UBA_HRCHY               ,T0.UBA_HRCHY_LO            ,T0.TRANS_CYCLE       ,T0.TRANS_RATE_CALC       ,T0.CUS_GROUP_NO       ,T1.SYS_TYPEfrom fdm_tmp.tmp_fsa_multi_path_funl_analyse_rslt_d_01_${hivevar:statis_date} t0   inner join FDM_DPA.FSA_MULTI_PATH_FUNL_VISIT_URL_HIS_D t1           on 1=1                                             and t0.comp_cond_type='10010201'  and t0.path_cond_type = '60020204'         and t0.UBA_HRCHY= '1' where t1.stat_date<='${statisdate}'and t1.stat_date>=t0.trans_cycle  and (t0.Page_Name = t1.page_name  or t1.page_id =t0.page_name)group by t1.acct_no                     ,t1.Page_ID                    ,t1.Page_Name                  ,t1.page_url                   ,t1.TRMNL_TYPE                     ,t1.Dev_ID           ,t0.path_id                        ,t0.UBA_HRCHY                ,t0.UBA_HRCHY_LO           ,t0.trans_cycle          ,t0.trans_rate_calc          ,T0.CUS_GROUP_NO          ,t1.SYS_TYPE;
```

**优化使用：配置如下参数，使用mapjoin替代common join.当然这里因为group by的原因还是会启动reduce进行去重。但是整体从4个小时优化到1.5小时。一般来说小表join大表一般配置下面四个参数就差不多，当然官方还提供了其他的参数共配置。[Hive官网参数配置](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FHive%2FConfiguration%2BProperties "Hive官网参数配置")**

1.  **set hive.auto.convert.join = true ;**   -- hive是否自动根据文件量大小，选择将common join转成map join 。hive 0.10 版本后的默认值 true。
2.  **set  hive.mapjoin.smalltable.filesize =25000000 ；**大表小表判断的阈值，如果表的大小小于该值25Mb，则会被判定为小表。则会被加载到内存中运行，将commonjoin转化成mapjoin。一般这个值也就最多几百兆的样子。
3.  **set  hive.auto.convert.join.noconditionaltask = true;**  翻译官网的解释：是否启用基于输入文件的大小，将普通连接转化为Map连接的优化机制。假设参与连接的表(或分区)有N个，如果打开这个参数，并且有N-1个表(或分区)的大小总和小于hive.auto.convert.join.noconditionaltask.size参数指定的值，那么会直接将连接转为Map连接。**（说人话：默认值：true，当将普通的join转化为普通的mapjoin时，是否将多个mapjoin转化为一个mapjoin，主要针对多个小表join大表的情形）**

![[Picture/ef817c029320078b51cf5199d0bf23b6_MD5.png]]

1.  **set  hive.auto.convert.join.noconditionaltask.size =10000000;** 翻译官网：如果hive.auto.convert.join.noconditionaltask是关闭的，则本参数不起作用。否则，如果参与连接的N个表(或分区)中的N-1个 的总大小小于这个参数的值，则直接将连接转为Map连接。默认值为10MB。**（说人话：将多个mapjoin转化为一个mapjoin时，其小表总和的最大值，所以这个条件比单独启动一个mapjon的参数****set  hive.mapjoin.smalltable.filesize更加严格。****）**

#### **尖叫提示：**

**1.一般遇到小表join大表，不管是多少个小表，把小表写在前面，开启mapjon，同时适当地调大上面的参数,Mapjoin几乎是解决小表join大表（包括笛卡尔乘积）的最好方式。尤其对于笛卡尔乘积的小表join大表来说，性能差别天壤之别。**

**2.所谓的mapjoin优化就是在Map阶段完成join工作，而不是像通常的common join在Reduce阶段按照join的列值进行分发数据到每个Reduce上进行join工作。前面我们知道，没有数据分发分布也就不会有数据倾斜的存在。实际上所谓的mapjoin并不是像有些人说的那样只是将小表加载到内存然后跟大表join那么简单，如果那样照样会有reduce的产生，也不会快那么多。而是会将所有的小表全量复制到每个map任务节点，然后再将小表缓存在每个map节点的内存里与大表进行join工作。所以这解释了为啥小表的大小的不能太大的原因，否则复制分发太多反而得不偿失。一般这个值也就几百兆吧。像我们公司每个map的分配的内存才2G，堆内存才1.5G，你要是搞个1个G的小表,直接很容易OOM报错了。**

**3.****在0.7.0版本之前：需要在sql中使用 /\*+ MAPJOIN(smallTable) \*/ 来开启mapjoin，而后则Hive会自动通过配置的参数来判断是否开启mapjoin。**

**4.对于小表join大表的笛卡尔乘积，还可以通过规避的方法避免：具体比如给 Join的两个表都增加一列Join key**，**原理很简单：将小表扩充一列join key，并将小表的总数复制数倍，join** **key 各不相同，比如第一次为1，复制一次joinkey为2，依次类推；将大表扩充一列join key 为随机数，这个随机数为小表里的joinkey的随机值，如1-5的随机值。这样就实现了将一个大表拆分几分同时处理，而且这样小表扩充了几倍，大表就被对应地分成几份处理。这种方式也可以提高笛卡尔乘积小表join大表的性能。**

### 2.笛卡尔乘积：大表join大表

       **大表join大表一般调优有四种方式具体参考其他博客，但是对于笛卡尔乘积来说，如果是小join大，开启mapjoin性能还不算太差，但要是大join大的笛卡尔乘积那是真可怕。**

**1.首先要尽量避免笛卡尔乘积，比如HQL无法支持循环，遍历等缺陷，这种情况遇到笛卡尔乘积的可以考虑用spark来替代，或者用UDF来解决，这是首选方案，其他几乎没有更好的处理方案了。**

优化的三种方式

在小表和大表进行join时，将**小表放在前边**，效率会高。hive会将小表进行缓存。

## 2、mapjoin

使用**mapjoin将小表放入内存**，在map端和大表逐一匹配。从而省去[reduce](https://so.csdn.net/so/search?q=reduce&spm=1001.2101.3001.7020)。

样例：

```
SELECTa.a1,a.a2,b.b2FROMtablea a JOIN tableb b ONa.a1 = b.b1
```

这里会有个问题，大表left join小表，大表会出现很多小表的字段，但是其中内容为NULL

记得过滤。

规范一点，应该小表left join大表（注意是left、right、inner），然后将小表加入mapjoin中。

> **map join 概念**：将其中做连接的小表（全量数据）分发到所有 MapTask 端进行 Join，从 而避免了 reduceTask，前提要求是内存足以装下该全量数据。 

-   使用map join解决小表关联大表造成的数据倾斜问题。这个方法使用的频率很高。
-   每个MapTask载入了大表的一个数据块做处理，载入小表的所有数据做处理，省去了ReduceTask，避免了分区不均，提高了效率。
-   大表放硬盘，小表放内存。

 [hive：使用map join解决大小表关联造成的数据倾斜\_dd1296的博客-CSDN博客](https://blog.csdn.net/weixin_41639302/article/details/107235828?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-1-107235828.pc_agg_new_rank&utm_term=%E5%A4%A7%E5%B0%8F%E8%A1%A8join%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E6%95%B0%E6%8D%AE%E5%80%BE%E6%96%9C&spm=1000.2123.3001.4430 "hive：使用map join解决大小表关联造成的数据倾斜_dd1296的博客-CSDN博客")

### 如果两张都是大表，能不能使用mapjoin？

可以。  
把其中一张大表切分成小表，然后分别 map join。（其实不太懂）

```
SELECT*FROMlog a LEFT JOIN(SELECTd.*FROM(SELECT DISTINCT user_id FROM log)cJOIN users dONc.user_id = d.user_id)x ON a.user_id = x.user_id;
```

## 3、set配置

（没啥用）在0.7版本号后。也能够用配置来自己主动优化

```
set hive.auto.convert.join=true;
```