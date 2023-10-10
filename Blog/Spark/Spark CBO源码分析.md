
## 一、背景

SDM中的Analyze Table和Auto Analyze已经完成，已经具备了根据hot path, hot table, hot column和hot partition进行自动Analyze和添加Analyze policy的功能，表的统计信息会越来约完善，因此个计算引擎可以根据这些统计信息优化plan，spark中根据表的统计信息优化plan就是CBO(cost based optimization)

## 二、Spark CBO

CBO是基于cost来优化plan，要计算cost就需要统计一些参与计算的表的相关信息，spark使用的统计信息包括表的统计信息Statistics和列的统计信息ColumnStat

CBO整体分为两部分：统计信息的获取和统计信息的使用

-   统计信息的获取

基础的统计信息由SDM中的Analyze模块使用presto生成，保留表和列的统计信息，保存至hive mate store中。spark各算子的统计信息由spark估算生成，可参考：
![[Picture/Pasted image 20230922102429.png]]

在unresolvedLogicalPlan->resolvedLogicalPlan过程中收集Statistics，然后在resolvedLogicalPlan->optimizedLogicalPlan过程中，基于这些统计信息，进行costBasedJoinRecorder，即基于统计信息，对join顺序重排序，寻求最优join方案。

在optimizedLogicalPlan→physicalPlan过程中，基于Statistics中的sizeInBytes信息以及hint选择合适的join策略(broadcastJoin, hashShuffledJoin, sortMergeJoin)

-   join type

join type的选取逻辑相对简单一些，可参考类：JoinSelection

-   join reorder

CostBasedJoinReorder是一个使用plan的stats信息，来选择合适的join顺序的类。

Optimizer类中有两个跟join 顺序有关的rule，一个是ReoderJoin，另外一个是CostBasedJoinRecorder。ReorderJoin是没有cbo也会触发的rule，这个不会使用统计的信息，只是负责将filter下推，这样最底层的join至少会有一个filter。如果这些join已经每个都有一条condition，那么这些plan就不会变化，因此reorder join不涉及基于代价的优化。

首先看下对cost的定义，cost包括card和size，card是基数也就是row count，size是大小

```
/**
 * This class defines the cost model for a plan.
 * @param card Cardinality (number of rows).
 * @param size Size in bytes.
 */
case class Cost(card: BigInt, size: BigInt) {
  def +(other: Cost): Cost = Cost(this.card + other.card, this.size + other.size)
}
```

CostBasedJoinReorder整体架构：

![[Picture/Pasted image 20230922102510.png]]

涉及参数：

-   spark.sql.cbo.enabled : 是否开启cbo，默认false
-   spark.sql.cbo.joinReorder.enabled : 是否开启join reorder，默认false
-   spark.sql.cbo.planStats.enabled : logical plan是否从catalog中fetch statistics
-   spark.sql.cbo.joinReorder.dp.threshold : join reorder允许的最大join节点数，默认12
-   spark.sql.cbo.starJoinFTRatio : start join 维度表允许是事实表大小的最大比率，默认0.9
-   spark.sql.cbo.joinReorder.card.weight : join reorder cost计算中基数的比重，默认0.7
-   spark.sql.cbo.joinReorder.dp.star.filter : join reorder时是否将star join先build在一起(不破坏star join)，默认false

源码分析：

如果没有开启cbo或者join reorder，plan将不做修改，否则会对Join或者Project->Join这两种plan进行reorder

```
def apply(plan: LogicalPlan): LogicalPlan = {
    if (!conf.cboEnabled || !conf.joinReorderEnabled) {
      plan // 没有开启cbo和join reorder
    } else {
      val result = plan transformDown {
        // Start reordering with a joinable item, which is an InnerLike join with conditions.
        // Avoid reordering if a join hint is present.
        case j @ Join(_, _, _: InnerLike, Some(cond), JoinHint.NONE) =>
          reorder(j, j.output)  // reorder Join
        case p @ Project(projectList, Join(_, _, _: InnerLike, Some(cond), JoinHint.NONE))
          if projectList.forall(_.isInstanceOf[Attribute]) =>
          reorder(p, p.output) // reorder Project->Join
      }
      // After reordering is finished, convert OrderedJoin back to Join.
      result transform {
        case OrderedJoin(left, right, jt, cond) => Join(left, right, jt, cond, JoinHint.NONE)
      }
    }
  }
```

进行reorder时，首先提取出plan中的InnerLike Join的各个child(items)和join condition，item的数量要在2和joinReorderDPThreshold(默认12)之间，join condition不为空，每个join item的rowCount都存在才可以进行reorder

```
private def reorder(plan: LogicalPlan, output: Seq[Attribute]): LogicalPlan = {
    val (items, conditions) = extractInnerJoins(plan)
    val result =
      // Do reordering if the number of items is appropriate and join conditions exist.
      // We also need to check if costs of all items can be evaluated.
      if (items.size > 2 && items.size <= conf.joinReorderDPThreshold && conditions.nonEmpty &&
          items.forall(_.stats.rowCount.isDefined)) {
        JoinReorderDP.search(conf, items, conditions, output)
      } else {
        plan
      }
    // Set consecutive join nodes ordered.
    replaceWithOrderedJoin(result)
  }
```

join reorder是采用动态规划算法寻找cost最小的plan的，foundPlans用于存储在动态规划过程中生成的各个plan，foundPlans中存储的是JoinPlanMap，其中可以是itemId的集合，value是对应的plan，item集合在foundPlans中是按层存储的，第k层存储的item数量是k+1，比如A J B J C J D，第0层存储的都是item为1的plan：p({A}), p({B}), p({C}), p({D})，第1层存储的都是item为2的plan：p({A, B}), p({B, C}), p({C, D})

buildJoinGraphInfo是将star join和non star join分开，不破坏star join，首先将star join build在一起

动态规划算法主要逻辑在searchLevel方法中

当level中只有一个plan时就不会继续想更上层的level优化了，只会继续优化本层的plan，最终剩余的plan就是最优的

```
def search(
      conf: SQLConf,
      items: Seq[LogicalPlan],
      conditions: ExpressionSet,
      output: Seq[Attribute]): LogicalPlan = {

    val startTime = System.nanoTime()
    // Level i maintains all found plans for i + 1 items.
    // Create the initial plans: each plan is a single item with zero cost.
    val itemIndex = items.zipWithIndex  // index 就是itemId, 用于识别是不是相同的join
    val foundPlans = mutable.Buffer[JoinPlanMap]({
      // SPARK-32687: Change to use `LinkedHashMap` to make sure that items are
      // inserted and iterated in the same order.
      val joinPlanMap = new JoinPlanMap
      itemIndex.foreach {
        case (item, id) =>
          joinPlanMap.put(Set(id), JoinPlan(Set(id), item, ExpressionSet(), Cost(0, 0)))
      }
      joinPlanMap
    })

    // Build filters from the join graph to be used by the search algorithm.
    val filters = JoinReorderDPFilters.buildJoinGraphInfo(conf, items, conditions, itemIndex) // 用于避免star join被分开

    // Build plans for next levels until the last level has only one plan. This plan contains
    // all items that can be joined, so there's no need to continue.
    val topOutputSet = AttributeSet(output)
    while (foundPlans.size < items.length) {
      // Build plans for the next level.
      foundPlans += searchLevel(foundPlans.toSeq, conf, conditions, topOutputSet, filters) // 动态规划寻找最优plan
    }

    val durationInMs = (System.nanoTime() - startTime) / (1000 * 1000)
    logDebug(s"Join reordering finished. Duration: $durationInMs ms, number of items: " +
      s"${items.length}, number of plans in memo: ${foundPlans.map(_.size).sum}")

    // The last level must have one and only one plan, because all items are joinable.
    assert(foundPlans.size == items.length && foundPlans.last.size == 1)
    foundPlans.last.head._2.plan match {
      case p @ Project(projectList, j: Join) if projectList != output =>
        assert(topOutputSet == p.outputSet)
        // Keep the same order of final output attributes.
        p.copy(projectList = output)
      case finalPlan if !sameOutput(finalPlan, output) =>
        Project(output, finalPlan)
      case finalPlan =>
        finalPlan
    }
  }
```

level0 就是各个join的child，existingLevels.length的大小是当前要计算的层(level是从level0开始算的)，level(existingLevels.length)中的item的个数是existingLevels.length+1，可使用的最大level是existingLevels.length-1，该层有existingLevels.length个plan，需要与level0组合才能生成有existingLevels.length+1个plan，因此2个层之和要等于existingLevels.length - 1才可以

```
/** Find all possible plans at the next level, based on existing levels. */
  private def searchLevel(
      existingLevels: Seq[JoinPlanMap],
      conf: SQLConf,
      conditions: ExpressionSet,
      topOutput: AttributeSet,
      filters: Option[JoinGraphInfo]): JoinPlanMap = {

    val nextLevel = new JoinPlanMap
    var k = 0
    val lev = existingLevels.length - 1
    // Build plans for the next level from plans at level k (one side of the join) and level
    // lev - k (the other side of the join).
    // For the lower level k, we only need to search from 0 to lev - k, because when building
    // a join from A and B, both A J B and B J A are handled.
    while (k <= lev - k) {
      val oneSideCandidates = existingLevels(k).values.toSeq
      for (i <- oneSideCandidates.indices) {
        val oneSidePlan = oneSideCandidates(i)
        val otherSideCandidates = if (k == lev - k) {
          // Both sides of a join are at the same level, no need to repeat for previous ones.
          oneSideCandidates.drop(i)
        } else {
          existingLevels(lev - k).values.toSeq
        }

        otherSideCandidates.foreach { otherSidePlan =>
          buildJoin(oneSidePlan, otherSidePlan, conf, conditions, topOutput, filters) match {
            case Some(newJoinPlan) =>
              // Check if it's the first plan for the item set, or it's a better plan than
              // the existing one due to lower cost.
              val existingPlan = nextLevel.get(newJoinPlan.itemIds)
              if (existingPlan.isEmpty || newJoinPlan.betterThan(existingPlan.get, conf)) { // 判断已存在的plan是否需要替换
                nextLevel.update(newJoinPlan.itemIds, newJoinPlan)
              }
            case None =>
          }
        }
      }
      k += 1
    }
    nextLevel
  }
```

A J B J C J D 的整体计算逻辑如下图：

![[Picture/Pasted image 20230922102728.png]]

build join就是根据根据join condition将join child组合起来，需要注意的一点是如果有star join，则不能把star join分开，join的两端不能有重合，且join condition 不能为空

```
private def buildJoin(
      oneJoinPlan: JoinPlan,
      otherJoinPlan: JoinPlan,
      conf: SQLConf,
      conditions: ExpressionSet,
      topOutput: AttributeSet,
      filters: Option[JoinGraphInfo]): Option[JoinPlan] = {

    if (oneJoinPlan.itemIds.intersect(otherJoinPlan.itemIds).nonEmpty) {
      // Should not join two overlapping item sets.
      return None
    }

    if (filters.isDefined) { // 有star join
      // Apply star-join filter, which ensures that tables in a star schema relationship
      // are planned together. The star-filter will eliminate joins among star and non-star
      // tables until the star joins are built. The following combinations are allowed:
      // 1. (oneJoinPlan U otherJoinPlan) is a subset of star-join
      // 2. star-join is a subset of (oneJoinPlan U otherJoinPlan)
      // 3. (oneJoinPlan U otherJoinPlan) is a subset of non star-join
      val isValidJoinCombination =
        JoinReorderDPFilters.starJoinFilter(oneJoinPlan.itemIds, otherJoinPlan.itemIds,
          filters.get) // 判断有没有将star join分开
      if (!isValidJoinCombination) return None
    }

    val onePlan = oneJoinPlan.plan
    val otherPlan = otherJoinPlan.plan
    val joinConds = conditions
      .filterNot(l => canEvaluate(l, onePlan))
      .filterNot(r => canEvaluate(r, otherPlan))
      .filter(e => e.references.subsetOf(onePlan.outputSet ++ otherPlan.outputSet))
    if (joinConds.isEmpty) {
      // Cartesian product is very expensive, so we exclude them from candidate plans.
      // This also significantly reduces the search space.
      return None
    }

    // Put the deeper side on the left, tend to build a left-deep tree.
    val (left, right) = if (oneJoinPlan.itemIds.size >= otherJoinPlan.itemIds.size) {
      (onePlan, otherPlan)
    } else {
      (otherPlan, onePlan)
    }
    val newJoin = Join(left, right, Inner, joinConds.reduceOption(And), JoinHint.NONE)
    val collectedJoinConds = joinConds ++ oneJoinPlan.joinConds ++ otherJoinPlan.joinConds
    val remainingConds = conditions -- collectedJoinConds
    val neededAttr = AttributeSet(remainingConds.flatMap(_.references)) ++ topOutput
    val neededFromNewJoin = newJoin.output.filter(neededAttr.contains)
    val newPlan =
      if ((newJoin.outputSet -- neededFromNewJoin).nonEmpty) {
        Project(neededFromNewJoin, newJoin)
      } else {
        newJoin
      }

    val itemIds = oneJoinPlan.itemIds.union(otherJoinPlan.itemIds)
    // Now the root node of onePlan/otherPlan becomes an intermediate join (if it's a non-leaf
    // item), so the cost of the new join should also include its own cost.
    val newPlanCost = oneJoinPlan.planCost + oneJoinPlan.rootCost(conf) +
      otherJoinPlan.planCost + otherJoinPlan.rootCost(conf)
    Some(JoinPlan(itemIds, newPlan, collectedJoinConds, newPlanCost))
  }

def starJoinFilter(
      oneSideJoinPlan: Set[Int],
      otherSideJoinPlan: Set[Int],
      filters: JoinGraphInfo) : Boolean = {
    val starJoins = filters.starJoins
    val nonStarJoins = filters.nonStarJoins
    val join = oneSideJoinPlan.union(otherSideJoinPlan)

    // Disjoint sets
    oneSideJoinPlan.intersect(otherSideJoinPlan).isEmpty &&
      // Either star or non-star is empty
      (starJoins.isEmpty || nonStarJoins.isEmpty ||
        // Join is a subset of the star-join
        join.subsetOf(starJoins) || // 该join是star join的子集
        // Star-join is a subset of join
        starJoins.subsetOf(join) || // star join是该join的子集
        // Join is a subset of non-star
        join.subsetOf(nonStarJoins)) // 没有star join
  }
```

判断plan的好坏是通过plan的cost计算的，joinReorderCardWeight是基数占的比重默认0.7

```
def betterThan(other: JoinPlan, conf: SQLConf): Boolean = {
      val thisCost = BigDecimal(this.planCost.card) * conf.joinReorderCardWeight +
        BigDecimal(this.planCost.size) * (1 - conf.joinReorderCardWeight)
      val otherCost = BigDecimal(other.planCost.card) * conf.joinReorderCardWeight +
        BigDecimal(other.planCost.size) * (1 - conf.joinReorderCardWeight)
      thisCost < otherCost
    }
```

find star join就是寻找一个事实表和若干个维度表，首先join的表要大于2且rowCount都存在，再将表按照size进行降序排序，事实表一般要比维度表大很多，检查表的大小是否符合这一条件，规则是维度表的大小要比事实表的starSchemaFTRatio(spark.sql.cbo.starJoinFTRatio，default：0.9)倍要小，否则就不是star join，如果满足条件就以最大的这张表为事实表，根据join condition匹配维度表。目前只支持以最大的表为事实表寻找star join，还不支持多个star join的情况

def findStarJoins( input: Seq\[LogicalPlan\], conditions: Seq\[Expression\]): Seq\[LogicalPlan\] = {

```
val emptyStarJoinPlan = Seq.empty[LogicalPlan]

if (input.size < 2) {
  emptyStarJoinPlan
} else {
  // Find if the input plans are eligible for star join detection.
  // An eligible plan is a base table access with valid statistics.
  val foundEligibleJoin = input.forall {
    case m if m.nodeName.equals("RangerSparkMasking") && m.stats.rowCount.isDefined => true // 兼容spark-ranger
    case r if r.nodeName.equals("RangerSparkRowFilter") && r.stats.rowCount.isDefined => true
    case PhysicalOperation(_, _, t: LeafNode) if t.stats.rowCount.isDefined => true
    case _ => false
  }

  if (!foundEligibleJoin) {
    // Some plans don't have stats or are complex plans. Conservatively,
    // return an empty star join. This restriction can be lifted
    // once statistics are propagated in the plan.
    emptyStarJoinPlan
  } else {
    // Find the fact table using cardinality based heuristics i.e.
    // the table with the largest number of rows.
    val sortedFactTables = input.map { plan =>
      TableAccessCardinality(plan, getTableAccessCardinality(plan))
    }.collect { case t @ TableAccessCardinality(_, Some(_)) =>
      t
    }.sortBy(_.size)(implicitly[Ordering[Option[BigInt]]].reverse) // 按照size降序

    sortedFactTables match {
      case Nil =>
        emptyStarJoinPlan
      case table1 :: table2 :: _
        if table2.size.get.toDouble > conf.starSchemaFTRatio * table1.size.get.toDouble =>
        // If the top largest tables have comparable number of rows, return an empty star plan.
        // This restriction will be lifted when the algorithm is generalized
        // to return multiple star plans.
        emptyStarJoinPlan // 最大的表不符合star join条件，没有发现star join
      case TableAccessCardinality(factTable, _) :: rest => // 发现事实表，寻找维度表
        // Find the fact table joins.
        // 略
```

}

## 三、社区相关优化

[https://issues.apache.org/jira/browse/SPARK-34120](https://issues.apache.org/jira/browse/SPARK-34120)

[https://issues.apache.org/jira/browse/SPARK-33959：](https://issues.apache.org/jira/browse/SPARK-33959%EF%BC%9A) Improve the statistics estimation of the Tail

优化Tail的Statistics的估计，原本tail的Statistics是直接继承child的，优化后会根据limit的数量计算size和rowCount

![[Picture/Pasted image 20230922103248.png]]

![[Picture/Pasted image 20230922103316.png]]

[https://issues.apache.org/jira/browse/SPARK-33954](https://issues.apache.org/jira/browse/SPARK-33954): Some operator missing rowCount when enable CBO

一些算子没有实现rowCount的估计，只有datasize，根据children的rowCount和size的乘积计算(SizeInBytesOnlyStatsPlanVisitor的默认计算逻辑就是相乘)

![[Picture/Pasted image 20230922103359.png]]
![[Picture/Pasted image 20230922103410.png]]
[https://issues.apache.org/jira/browse/SPARK-34031：Union](https://issues.apache.org/jira/browse/SPARK-34031%EF%BC%9AUnion) operator missing rowCount when enable CBO

union实现rowCount和size，即：children之和

![[Picture/Pasted image 20230922103427.png]]
![[Picture/Pasted image 20230922103437.png]]

[https://issues.apache.org/jira/browse/SPARK-33956：Add](https://issues.apache.org/jira/browse/SPARK-33956%EF%BC%9AAdd) rowCount for Range operator

range实现rowCount，element的数量

![[Picture/Pasted image 20230922103459.png]]
![[Picture/Pasted image 20230922103506.png]]
[https://issues.apache.org/jira/browse/SPARK-34119：Keep](https://issues.apache.org/jira/browse/SPARK-34119%EF%BC%9AKeep) necessary stats after partition pruning

优化分区裁剪后的stat

![[Picture/Pasted image 20230922103517.png]]

![[Picture/Pasted image 20230922103527.png]]

[https://issues.apache.org/jira/browse/SPARK-34121：Intersect](https://issues.apache.org/jira/browse/SPARK-34121%EF%BC%9AIntersect) operator missing rowCount when CBO enabled

优化Intersect算子的rowCount和size，根据size两者取其小

![[Picture/Pasted image 20230922103537.png]]

[https://issues.apache.org/jira/browse/SPARK-35185：Improve](https://issues.apache.org/jira/browse/SPARK-35185%EF%BC%9AImprove) Distinct statistics estimation

优化distinct的Statistics，用output就行聚合

![[Picture/Pasted image 20230922103551.png]]

[https://issues.apache.org/jira/browse/SPARK-35203：Improve](https://issues.apache.org/jira/browse/SPARK-35203%EF%BC%9AImprove) Repartition statistics estimation

优化Repartition算子的Statistics，使用其child的stat，使用fallback只是在BasicStatsPlanVisitor和SizeInBytesOnlyStatsPlanVisitor计算逻辑相同时易于维护

![[Picture/Pasted image 20230922103600.png]]

![[Picture/Pasted image 20230922103607.png]]