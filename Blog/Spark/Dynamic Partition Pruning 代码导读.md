Git issue [https://issues.apache.org/jira/browse/SPARK-11150](https://issues.apache.org/jira/browse/SPARK-11150)  ，DPP 背景这里就不做过多介绍，可以参考 issue 描述。

#### 具体实现

DPP 整体的思路是通过在 join 两表中的一侧插入一个 filter subquery 用来过滤数据，从而减少参与 join 的数据量。

可以先了解下 spark 内的 subquery 机制，Scalar subquery 和 Predicate subquery [https://databricks-prod-cloudfront.cloud.databricks.com/public/4027ec902e239c93eaaa8714f173bcfc/2728434780191932/1483312212640900/6987336228780374/latest.html。](https://databricks-prod-cloudfront.cloud.databricks.com/public/4027ec902e239c93eaaa8714f173bcfc/2728434780191932/1483312212640900/6987336228780374/latest.html%E3%80%82)

#### Optimize 阶段

在 logicalPlan 优化阶段，具体逻辑在 `PartitionPruning` 这个rule，入口即 apply 方法，初步进行了一些判断，subquery 和 DPP 的参数没有开启时，都不会做处理。其余的情况则会进入 `prune` 方法。

![[Picture/Pasted image 20230925113746.png]]

在 `prune` 方法中，遍历整个 logicalPlan 找到所有 join 节点，对每个 join 节点，都会进行以下处理逻辑。首先从 join 中获取 equi-join key, 后面会用到

![[Picture/Pasted image 20230925113755.png]]

然后，使用 `splitConjunctivePredicates` 打平所有的通过 AND 连接的 join condition,  比如，“key1 = key2 AND key1 = key3”， 如果是“key1 = key2 OR key1 = key3”， 这种情况应该是无法适用 DPP 的。

对于所有 `EqualTo` 的表达式，比如 “key1 = key2”， 会按照 left → right 的顺序生成 kv 。

![[Picture/Pasted image 20230925113807.png]]

接下来首先判断左表是否为 partition table，只有 partition table 才可能被裁减，然后是 `canPruneLeft` 方法，Inner/LeftSemi/RightOuter 三种 join 类型时左表可以被裁减，这里需要理解一下。最后一个判断条件 `hasPartitionPruningFilter(right)`，判断右表是否有 `SelectivePredicate`，具体实现在 `isLikelySelective` 方法，也就是右表是否有过滤条件（LIKE, IN, contains 等等）。如果左表不符合以上三个条件，则判断右表是否可以被裁减，判断的逻辑和左表一样，注意 `canPruneRight` 方法，Inner/LeftSemi/RightOuter 三种 join 类型时右表可以被裁减。

左表或右表符合条件后，处理也比较简单，通过 `insertPredicate` 方法，插入 Filter 节点，Filter 节点的 condition 为 `DynamicPruningSubquery`，用来标识这是一个 DPP 的 filter。这样遍历所有 EqualTo 表达式，符合条件就在左右表的 logicalPlan 头上加 Filter，最后用新的 newLeft 和 newRight，构造出新的 Join 节点替换掉原来的 Join 节点。

![[Picture/Pasted image 20230925113823.png]]

还有一个比较重要的方法，`pruningHasBenefit()`，用来判断对 partition table 的裁剪是否有收益，对 partition table 裁剪过滤掉的数据量是否大于扫描另一张表的数据量，主要是在无法 reuse  broadcast exchange 时，是否值得增加一个 subquery。首先定义一个 ratio，根据 cbo column statistics 估算，默认值为0.5，通过配置 `spark.sql.optimizer.dynamicPartitionPruning.fallbackFilterRatio` 指定，即默认认为这次裁剪可以过滤掉 partition table 50%的数据量。

#### Physical execution

在  sparkPlan preparation 阶段，`PlanDynamicPruningFilters` rule 会负责改写执行计划。`apply` 方法首先会判断 DPP 配置是否打开。然后会遍历 sparkPlan 找到 `DynamicPruningSubquery` 节点，然后判断是否有可重用的 broadcast exchange

![[Picture/Pasted image 20230925113912.png]]

不论是否有可重用的 exchange, 都会构造一个 `DynamicPruningExpression`，只是该表达式的 child 会有所不同。如果有可重用的 exchange，则 child 为 `InSubqueryExec`（专门用于 DPP），`InSubqueryExec` 的 child 为由可重用的 broadcast exchange 转换得来的 `SubqueryBroadcastExec`，用来计算维表的 distinct join-key。如果没有可重用的 exchange 且 之前方法 `pruningHasBenefit` 为false，则 child 为一个true常量表达式，此时不会有分区裁剪。否则，会插入一个 `InSubquery` 表达式，随后通过 `PlanSubqueries` rule，会把该节点替换为 `InSubqueryExec`， `InSubqueryExec` 的 child 是一个 aggregate 算子，其实是使用 aggregate 计算维表的 distinct join-key。无论是可重用的 `subqueryBroadcastExec` 还是额外增加的 `subqueryExec`，在 SparkPlan 的 `prepare()` 和 `waitForSubqueries()` 阶段，都会 collect 维表 join 字段的值，然后 broadcast 出去，供后续使用。

![[Picture/Pasted image 20230925113920.png]]

在 `FileSourceScanExec` 中，计算 `dynamicallySelectedPartitions`(lazy，create rdd 时才会触发计算)，会对 `selectedPartitions`(静态的裁剪后的分区列表)根据 `InSubqueryExec` 进行过滤，生成最后的经过动态裁剪后的分区列表，创建 rdd 时会指定这些分区。

#### Support DPP in Adaptive Query Execution

Spark 3.2之前，一旦在`logicalPlan`优化阶段通过`PartitionPruning`插入了`DynamicPruningSubquery`算子，对应物理计划则无法插入 `AdaptiveSparkPlanExec`算子，即无法应用 AQE,

```
    // TODO migrate dynamic-partition-pruning onto adaptive execution.
    sanityCheck(plan) &&
      !plan.logicalLink.exists(_.isStreaming) &&
      !plan.expressions.exists(_.find(_.isInstanceOf[DynamicPruningSubquery]).isDefined) &&
    plan.children.forall(supportAdaptive)
  }
```

而且 AQE 的优化规则中也没有对 DPP 的相关优化。 Spark 3.2 对 AQE 和 DPP 的兼容进行了支持，主要通过`PlanAdaptiveDynamicPruningFilters`完成，具体逻辑与`PlanDynamicPruningFilters`类似。

首先，在`InsertAdaptiveSparkPlan`的`PlanAdaptiveSubqueries`阶段，将逻辑计划优化阶段生成的`DynamicPruningSubquery`替换为`DynamicPruningExpression`，在`AdaptiveSparkPlanExec`的`queryStageOptimizerRules`中加入`PlanAdaptiveDynamicPruningFilters`，主要负责对前一阶段插入的`DynamicPruningExpression`进行处理。如果有可重用的`BroadcastExchange`，则将`DynamicPruningExpression`中原先的`SubqueryAdaptiveBroadcastExec`替换为可执行的`SubqueryBroadcastExec`。其余逻辑也与`PlanDynamicPruningFilters`类似。

```
def apply(plan: SparkPlan): SparkPlan = {
    if (!conf.dynamicPartitionPruningEnabled) {
      return plan
    }

    plan transformAllExpressions {
      case DynamicPruningExpression(InSubqueryExec(
          value, SubqueryAdaptiveBroadcastExec(name, index, onlyInBroadcast, buildPlan, buildKeys,
          adaptivePlan: AdaptiveSparkPlanExec), exprId, _)) =>
        val packedKeys = BindReferences.bindReferences(
          HashJoin.rewriteKeyExpr(buildKeys), adaptivePlan.executedPlan.output)
        val mode = HashedRelationBroadcastMode(packedKeys)
        // plan a broadcast exchange of the build side of the join
        val exchange = BroadcastExchangeExec(mode, adaptivePlan.executedPlan)

        val canReuseExchange = conf.exchangeReuseEnabled && buildKeys.nonEmpty &&
          find(rootPlan) {
            case BroadcastHashJoinExec(_, _, _, BuildLeft, _, left, _, _) =>
              left.sameResult(exchange)
            case BroadcastHashJoinExec(_, _, _, BuildRight, _, _, right, _) =>
              right.sameResult(exchange)
            case _ => false
          }.isDefined

        if (canReuseExchange) {
          exchange.setLogicalLink(adaptivePlan.executedPlan.logicalLink.get)
          val newAdaptivePlan = adaptivePlan.copy(inputPlan = exchange)

          val broadcastValues = SubqueryBroadcastExec(
            name, index, buildKeys, newAdaptivePlan)
          DynamicPruningExpression(InSubqueryExec(value, broadcastValues, exprId))
        } else if (onlyInBroadcast) {
          DynamicPruningExpression(Literal.TrueLiteral)
        } else {
          // we need to apply an aggregate on the buildPlan in order to be column pruned
          val alias = Alias(buildKeys(index), buildKeys(index).toString)()
          val aggregate = Aggregate(Seq(alias), Seq(alias), buildPlan)

          val session = adaptivePlan.context.session
          val planner = session.sessionState.planner
          // Here we can't call the QueryExecution.prepareExecutedPlan() method to
          // get the sparkPlan as Non-AQE use case, which will cause the physical
          // plan optimization rules be inserted twice, once in AQE framework and
          // another in prepareExecutedPlan() method.
          val sparkPlan = QueryExecution.createSparkPlan(session, planner, aggregate)
          val newAdaptivePlan = adaptivePlan.copy(inputPlan = sparkPlan)
          val values = SubqueryExec(name, newAdaptivePlan)
          DynamicPruningExpression(InSubqueryExec(value, values, exprId))
        }
    }
```