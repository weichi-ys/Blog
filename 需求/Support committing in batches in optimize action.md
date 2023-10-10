实现目标：
	小文件合并
	文件间排序
	文件内排序

背景：
	目前支持的优化都是在最新粒度的分区上面，一个分区的优化就会导致一个commit，同时为了避免commit并发执行过程中的相互影响，采用的是串行的方式执行

注意点：
	一个文件的修改会产生一个commit，同个commit在并发执行的时候，会出现冲突的问题。导致数据丢失。
	合并小文件时会出现多个commit，这些commit在执行的时候，可能与用户的ETL产生的commit之间产生竞争关系，导致这个优化措施影响到线上ETL的任务运行。

提高的解决方案：
	在commit的时候，先将commit进行合并，将小的commit合并成一个大的，然后将合并后的比较大的commit做最后的commit，减少commit的数量。

准备工作：
	熟悉：BaseOptimizeTableAction
	测试：TestOptimizeTableAction

代码跟踪：
	文件内排序：testOptimizeSingleSortedFile
	文件间排序：
	小文件合并：testMergeSmallFiles

解决方案：
1. 之前的方案是一个partition生成一个优化作业，其中在analyze()方法中，有以下的代码：
```
if (partitionsThreshold > 0 && partitionToBytes.size() > partitionsThreshold) {  
  throw new RuntimeException(String.format(  
      "Too much partitions(=%d) to be optimized, which exceeds the threshold %d!",  
      partitionToBytes.size(), partitionsThreshold));  
}
``` 
	此时，当partitionToBytes.size的数量远远小于partitionThreshold时，其实会可以将多个parttionToBytes的数据进行结合，生成一个新的文件。
	注意点：合并小文件时，要求一个partition在一起，或者相邻的partition数据结合在一起
2. 实现时，需要注意每一个partition仍需要一个snapshot，还是说在多个partition数据进行处理的时候，将多个partition的作业产生的addFile和deleteFile放在一起，同时采用一个snapshot序列号即可？





sourceTable：
```
CREATE TABLE source_table (
	 a INT
	,b INT
	,c STRING
	,d STRING
	,arrayData array<int>
	,mapData map<string, long>
	,jsonData string
	) USING iceberg PARTITIONED BY (d)
```


优化Action的代码逻辑
1. BaseOptimizeTableAction中的execute()方法；
2. 在execute()方法中调用OptimizeContext对象的analyze生成相应的调度任务。此时，每个partition生成一个优化作业，并存放在一个Map之中。具体看下面代码：
```
partitionToJob.put(  
    part,  
    OptimizePartitionJob.newJob(newJobId(), new Input(part, files, bytes),  
        parallelism));
```
3. 在execute()方法中继续向下运行，调用doExecute()方法；
4. 在doExecute()方法中执行doOverWrite方法；
5. 在doOverWrite()方法中将第2步中生成的作业进行调度，执行相应的数据改变逻辑；
6. doExecute()方法继续运行，得到最终的snapshot提交
7. 



流程：
org.apache.iceberg.actions#execute()
