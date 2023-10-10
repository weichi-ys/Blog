#### Background and Motivation

[https://issues.apache.org/jira/browse/SPARK-32268](https://issues.apache.org/jira/browse/SPARK-32268)

DPP 通过对大表直接进行 partition 级别的裁剪，可以大大提高查询速度，但 DPP 的适用条件也相对严格，需要大表的分区列参与 join，但如果大表参与 join 的列为非分区列则无法应用。我们知道 shuffle 是比较耗时的操作，shuffle 的数据量越大，耗时越久，而且对网络，机器IO都会产生比较大的压力。如果能在大表 shuffle 前根据非分区列的 join 列对其进行过滤，即使无法像 DPP 一样直接减少从存储中读取的数据量，但减小了其参与 shuffle 以及后续操作的数据量，可能也会得到比较不错的收益，这就是 dynamic bloom filter 的动机，即运行时预先扫描小表获取 join 列的值，构造 bloom filter 对大表进行过滤。

Bloom filter 是一种空间高效的数据结构，具体原理可以参考： [https://blog.csdn.net/jiaomeng/article/details/1495500](https://blog.csdn.net/jiaomeng/article/details/1495500)

#### Implement

Dynamic bloom filter 的实现思路和 DPP 基本一致。

##### Optimize LogicalPlan

首先在`SparkOptimizer`新增了规则`DynamicBloomFilterPruning`，逻辑与`PartitionPruning`类似，符合一系列判断条件之后插入节点`DynamicBloomFilterPruningSubquery`。与 DPP 不同的是，如果 join 可以被转化为 BroadcastHashJoin，则不会应用该规则，因为在 BroadcastHashJoin 的情况下对大表进行预先的过滤其实是多余的（非pushdown的情况下）。

```
case j @ Join(left, right, joinType, Some(condition), hint)
          if !canPlanAsBroadcastHashJoin(j, conf)
```

判断是否加入 filter 节点的主要逻辑如下，这里以裁剪左表（左右两侧都为 logicalPlan，为了方便表达，用左右表指代）为例进行说明，需要满足以下条件：

-   右表 rowCount 需要小于左表
-   Join 类型支持裁剪左表
-   右表 rowCount > 0
-   右表 rowCount 小于 `spark.sql.optimizer.dynamicBloomFilterJoinPruning.maxBloomFilterEntries`，默认值为100000000，避免 bloom filter 占用内存过大
-   右表中没有`DynamicBloomFilterPruningSubquery`
-   右表不是 stream 且存在 `SelectivePredicate`
-   左表(这里的左表是真正的左表或者包含左表的`Filter`节点)没有没有`SelectivePredicate`，因为如果存在`SelectivePredicate`，那么下一步便无法根据统计信息去计算过滤收益
-   使用`pruningHasBenefit`方法计算过滤是否有收益

```
            if (isRightSideSmall && canPruneLeft(joinType) && rightRowCnt > 0 &&
              rightRowCnt <= conf.dynamicBloomFilterJoinPruningMaxBloomFilterEntries &&
              !hasDynamicBloomFilterPruningSubquery(right) && supportDynamicPruning(right) &&
              getFilterableTableScan(l, left).exists { p =>
                !hasSelectivePredicate(p) &&
                  pruningHasBenefit(rightRowCnt, rightDistCnt, leftDistCnt, p)
              }) {
              newLeft = insertPredicate(l, newLeft, r, right, rightKeys, rightDistCnt, rightRowCnt)
              newHint = newHint.copy(leftHint = Some(HintInfo(strategy = Some(NO_BROADCAST_HASH))))
            }
```

满足以上条件会插入`DynamicBloomFilterPruningSubquery`节点，其中构建 BloomFilter 时也做到了 SortMergeJoin 另一侧 exchange 节点的复用，见代码

```
    val expectedNumItems = distinctCnt.getOrElse(rowCount)
    val coalescePartitions = math.max(math.ceil(expectedNumItems.toDouble / 4000000.0).toInt, 1)
    // Use EnsureRequirements shuffle origin to reuse the exchange.
    val repartition = RepartitionByExpression(joinKeys, filteringPlan,
      optNumPartitions = Some(conf.numShufflePartitions),
      withEnsureRequirementsShuffleOrigin = true)
    // Coalesce partitions to improve build bloom filter performance.
    val coalesce = Repartition(coalescePartitions, shuffle = false, repartition)
    val bloomFilter = Aggregate(Nil,
      Seq(BuildBloomFilter(filteringKey, expectedNumItems.toLong, distinctCnt.isEmpty, 0, 0)
        .toAggregateExpression()).map(e => Alias(e, e.sql)()), coalesce)
```

##### Execution

在 prepare 阶段，`PlanAdaptiveSubqueries`会把`DynamicBloomFilterPruningSubquery`节点替换为`DynamicPruningExpression(InBloomFilterSubqueryExec(_, _, _))`，扩展了`PlanAdaptiveDynamicPruningFilters`，支持对以上节点进行处理。新增了`BuildBloomFilter`和`InBloomFilter`两个 UDF。`BuildBloomFilter`在 sparkPlan prepare 阶段提交任务构造`BloomFilter`并 broadcast 出去，具体的 evaluate 逻辑还是交给`InBloomFilter`。

另外在 AQE 的`reOptimize`阶段也新增了规则`OptimizeBloomFilterJoin`，这个规则主要是用来根据执行过程的 metric 信息更新`BuildBloomFilter`的`expectedNumItems`。

##### Dynamic bloom filter 兼容 DPP

首先在`SparkOptimizer`中，两条优化规则的顺序如下

```
    Batch("PartitionPruning", Once,
      PartitionPruning) :+
    Batch("Pushdown Filters from PartitionPruning", fixedPoint,
      PushDownPredicates) :+
    Batch("BloomFilterPruning", Once, DynamicBloomFilterPruning) :+
    Batch("Pushdown Filters from BloomFilterPruning", fixedPoint,
      PushDownPredicates) :+
```

`PartitionPruning`规则优化后，所插入的`DynamicPruningSubquery`节点会被下推到`FileSourceScan`中，然后再进行`BloomFilterPruning`优化，所以`DynamicBloomFilterPruningSubquery`中是会包含`DynamicPruningSubquery`，因此在物理计划阶段，遇到`DynamicBloomFilterPruningSubquery`节点也需要考虑兼容DPP。

```
      case DynamicPruningExpression(i @ InBloomFilterSubqueryExec(
        _, SubqueryAdaptiveShuffleExec(name, buildPlan, _,
          adaptivePlan: AdaptiveSparkPlanExec), _, _)) =>
        // Used to resolve the nested DPP is inside the InBloomFilterSubquery.
        val subqueryMap = mutable.HashMap.empty[Long, DynamicPruningExpression]
        adaptivePlan.inputPlan.collectLeaves().foreach { f =>
          f.expressions.foreach {
            case DynamicPruningExpression(InSubqueryExec(
            value, SubqueryAdaptiveBroadcastExec(name, index, onlyInBroadcast, buildPlan, buildKeys,
            adaptivePlan: AdaptiveSparkPlanExec), exprId, _)) =>
              subqueryMap.put(exprId.id,
                planDynamicPartitionPruning(
                  value, name, index, onlyInBroadcast, buildPlan, buildKeys, adaptivePlan, exprId))
            case _ =>
          }
        }
```