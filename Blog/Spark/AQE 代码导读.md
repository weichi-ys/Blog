AQE实现的社区相关文档 [https://docs.google.com/document/d/1mpVjvQZRAkD-Ggy6-hcjXtBPiQoVbZGe3dLnAKgtJ4k/edit?usp=sharing](https://docs.google.com/document/d/1mpVjvQZRAkD-Ggy6-hcjXtBPiQoVbZGe3dLnAKgtJ4k/edit?usp=sharing)

社区issue [https://issues.apache.org/jira/browse/SPARK-31412](https://issues.apache.org/jira/browse/SPARK-31412)

#### 实现思路

![[Picture/Pasted image 20230925113324.png]]

将SparkPlan通过Exchange切分成多个QueryStage, QueryStage4 依赖QueryStage1、QueryStage2、 QueryStage3, 当执行SparkPlan的时候会判断QueryStage4依赖是否完成，没有完成的会递归调度依赖的QueryStage 执行, 当QueryStage1、QueryStage2、QueryStage3执行完成后，QueryStage4依赖runtime数据优化QueryStage4 的执行计划。

#### 具体实现

一切从调用`QueryExecution.executedPlan` 开始

```
  lazy val executedPlan: SparkPlan = {
    assertOptimized()
    executePhase(QueryPlanningTracker.PLANNING) {
      QueryExecution.prepareForExecution(preparations, sparkPlan.clone())
    }
  }
```

这个方法的核心就是`QueryExecution.prepareForExecution` 将各种策略应用在`optimizedPlan` 上

```
  private[execution] def prepareForExecution(
      preparations: Seq[Rule[SparkPlan]],
      plan: SparkPlan): SparkPlan = {
    val planChangeLogger = new PlanChangeLogger[SparkPlan]()
    val preparedPlan = preparations.foldLeft(plan) { case (sp, rule) =>
      val result = rule.apply(sp)
      planChangeLogger.logRule(rule.ruleName, sp, result)
      result
    }
    planChangeLogger.logBatch("Preparations", plan, preparedPlan)
    preparedPlan
  }
```

具体的策略如下

```
  private[execution] def preparations(
      sparkSession: SparkSession,
      adaptiveExecutionRule: Option[InsertAdaptiveSparkPlan] = None): Seq[Rule[SparkPlan]] = {
    // `AdaptiveSparkPlanExec` is a leaf node. If inserted, all the following rules will be no-op
    // as the original plan is hidden behind `AdaptiveSparkPlanExec`.
    adaptiveExecutionRule.toSeq ++
    Seq(
      CoalesceBucketsInJoin,
      PlanDynamicPruningFilters(sparkSession),
      PlanSubqueries(sparkSession),
      RemoveRedundantProjects,
      EnsureRequirements,
      // `RemoveRedundantSorts` needs to be added after `EnsureRequirements` to guarantee the same
      // number of partitions when instantiating PartitioningCollection.
      RemoveRedundantSorts,
      DisableUnnecessaryBucketedScan,
      ApplyColumnarRulesAndInsertTransitions(sparkSession.sessionState.columnarRules),
      CollapseCodegenStages(),
      ReuseExchange,
      ReuseSubquery
    )
  }
```

AQE的关键就是`InsertAdaptiveSparkPlan` 这个`Rule`, 这个规则当AQE生效时会把最上层的SparkPlan包装成`AdaptiveSparkPlanExec`, 这个类的特质是 `LeafExecNode` 因此AQE生效后后面的那些规则的生效逻辑就在`AdaptiveSparkPlanExec` 里面了，下面先进入`InsertAdaptiveSparkPlan`里面的逻辑

```
override def apply(plan: SparkPlan): SparkPlan = applyInternal(plan, false)

  private def applyInternal(plan: SparkPlan, isSubquery: Boolean): SparkPlan = plan match {
    case _ if !conf.adaptiveExecutionEnabled => plan
    case _: ExecutedCommandExec => plan
    case c: DataWritingCommandExec => c.copy(child = apply(c.child))
    case c: V2CommandExec => c.withNewChildren(c.children.map(apply))
    case _ if shouldApplyAQE(plan, isSubquery) =>
      if (supportAdaptive(plan)) {
        try {
          // Plan sub-queries recursively and pass in the shared stage cache for exchange reuse.
          // Fall back to non-AQE mode if AQE is not supported in any of the sub-queries.
          val subqueryMap = buildSubqueryMap(plan)
          val planSubqueriesRule = PlanAdaptiveSubqueries(subqueryMap)
          val preprocessingRules = Seq(
            planSubqueriesRule)
          // Run pre-processing rules.
          val newPlan = AdaptiveSparkPlanExec.applyPhysicalRules(plan, preprocessingRules)
          logDebug(s"Adaptive execution enabled for plan: $plan")
          AdaptiveSparkPlanExec(newPlan, adaptiveExecutionContext, preprocessingRules, isSubquery)
        } catch {
          case SubqueryAdaptiveNotSupportedException(subquery) =>
            logWarning(s"${SQLConf.ADAPTIVE_EXECUTION_ENABLED.key} is enabled " +
              s"but is not supported for sub-query: $subquery.")
            plan
        }
      } else {
        logWarning(s"${SQLConf.ADAPTIVE_EXECUTION_ENABLED.key} is enabled " +
          s"but is not supported for query: $plan.")
        plan
      }

    case _ => plan
  }
```

核心逻辑就是看是不是能应用AQE, 对于`!conf.adaptiveExecutionEnabled` `ExecutedCommandExec` 直接返回plan不用`AdaptiveSparkPlanExec`这个类包装，对于`DataWritingCommandExec` `V2CommandExec` 则对子节点递归调用 `InsertAdaptiveSparkPlan` 这个`Rule` 其他的就通过 `shouldApplyAQE(plan, isSubquery)` 来判断是否开启AQE，看下具体逻辑

```
  // AQE is only useful when the query has exchanges or sub-queries. This method returns true if
  // one of the following conditions is satisfied:
  //   - The config ADAPTIVE_EXECUTION_FORCE_APPLY is true.
  //   - The input query is from a sub-query. When this happens, it means we've already decided to
  //     apply AQE for the main query and we must continue to do it.
  //   - The query contains exchanges.
  //   - The query may need to add exchanges. It's an overkill to run `EnsureRequirements` here, so
  //     we just check `SparkPlan.requiredChildDistribution` and see if it's possible that the
  //     the query needs to add exchanges later.
  //   - The query contains sub-query.
  private def shouldApplyAQE(plan: SparkPlan, isSubquery: Boolean): Boolean = {
    conf.getConf(SQLConf.ADAPTIVE_EXECUTION_FORCE_APPLY) || isSubquery || {
      plan.find {
        case _: Exchange => true
        case p if !p.requiredChildDistribution.forall(_ == UnspecifiedDistribution) => true
        case p => p.expressions.exists(_.find {
          case _: SubqueryExpression => true
          case _ => false
        }.isDefined)
      }.isDefined
    }
  }
```

如下情况下可以开启AQE，1.强制开启AQE 2.改plan是子查询 3.存在Exchange 4.存在非UnspecifiedDistribution分布需求 5.存在子查询 接下来就是判断plan是否支持应用AQE，具体逻辑

```
  private def supportAdaptive(plan: SparkPlan): Boolean = {
    // TODO migrate dynamic-partition-pruning onto adaptive execution.
    sanityCheck(plan) &&
      !plan.logicalLink.exists(_.isStreaming) &&
      !plan.expressions.exists(_.find(_.isInstanceOf[DynamicPruningSubquery]).isDefined) &&
    plan.children.forall(supportAdaptive)
  }

  private def sanityCheck(plan: SparkPlan): Boolean =
    plan.logicalLink.isDefined
```

逻辑计划中不存在 `Streaming` 和 `DynamicPruningSubquery` 就可以应用AQE。具体的应用逻辑从`buildSubqueryMap`开始

```
   private def buildSubqueryMap(plan: SparkPlan): Map[Long, SubqueryExec] = {
    val subqueryMap = mutable.HashMap.empty[Long, SubqueryExec]
    plan.foreach(_.expressions.foreach(_.foreach {
      case expressions.ScalarSubquery(p, _, exprId)
          if !subqueryMap.contains(exprId.id) =>
        val executedPlan = compileSubquery(p)
        verifyAdaptivePlan(executedPlan, p)
        val subquery = SubqueryExec.createForScalarSubquery(
          s"subquery#${exprId.id}", executedPlan)
        subqueryMap.put(exprId.id, subquery)
      case expressions.InSubquery(_, ListQuery(query, _, exprId, _))
          if !subqueryMap.contains(exprId.id) =>
        val executedPlan = compileSubquery(query)
        verifyAdaptivePlan(executedPlan, query)
        val subquery = SubqueryExec(s"subquery#${exprId.id}", executedPlan)
        subqueryMap.put(exprId.id, subquery)
      case _ =>
    }))

    subqueryMap.toMap
  }
```

递归所有的expression，如果表达式有子查询就对子查询也应用`InsertAdaptiveSparkPlan`规则，注意这里公用了 `adaptiveExecutionContext`, 最终效果`buildSubqueryMap` 返回 `key: expression-id` `value:SparkPlan`,这个子`SparkPlan` 一定会被`AdaptiveSparkPlanExec`包住，接着构建规则`PlanAdaptiveSubqueries(subqueryMap)` 这个规则主要就是对表达式中的子查询进行复用的具体如下

```
def apply(plan: SparkPlan): SparkPlan = {
    plan.transformAllExpressions {
      case expressions.ScalarSubquery(_, _, exprId) =>
        execution.ScalarSubquery(subqueryMap(exprId.id), exprId)
      case expressions.InSubquery(values, ListQuery(_, _, exprId, _)) =>
        val expr = if (values.length == 1) {
          values.head
        } else {
          CreateNamedStruct(
            values.zipWithIndex.flatMap { case (v, index) =>
              Seq(Literal(s"col_$index"), v)
            }
          )
        }
        InSubqueryExec(expr, subqueryMap(exprId.id), exprId)
    }
  }
```

接着就是先来一把子查询复用，后面就将最初的plan包装成了`AdaptiveSparkPlanExec` ，`AdaptiveSparkPlanExec` 对外暴露为SparkPlan通过`doExecute`触发核心方法 `getFinalPhysicalPlan` 下面就进入`AdaptiveSparkPlanExec` 看看里面怎么实现的

```
  private def getFinalPhysicalPlan(): SparkPlan = lock.synchronized {
    if (isFinalPlan) return currentPhysicalPlan

    // In case of this adaptive plan being executed out of `withActive` scoped functions, e.g.,
    // `plan.queryExecution.rdd`, we need to set active session here as new plan nodes can be
    // created in the middle of the execution.
    context.session.withActive {
      val executionId = getExecutionId
      var currentLogicalPlan = currentPhysicalPlan.logicalLink.get
      var result = createQueryStages(currentPhysicalPlan)
      val events = new LinkedBlockingQueue[StageMaterializationEvent]()
      val errors = new mutable.ArrayBuffer[Throwable]()
      var stagesToReplace = Seq.empty[QueryStageExec]
      while (!result.allChildStagesMaterialized) {
        currentPhysicalPlan = result.newPlan
        if (result.newStages.nonEmpty) {
          stagesToReplace = result.newStages ++ stagesToReplace
          executionId.foreach(onUpdatePlan(_, result.newStages.map(_.plan)))

          // Start materialization of all new stages and fail fast if any stages failed eagerly
          result.newStages.foreach { stage =>
            try {
              stage.materialize().onComplete { res =>
                if (res.isSuccess) {
                  events.offer(StageSuccess(stage, res.get))
                } else {
                  events.offer(StageFailure(stage, res.failed.get))
                }
              }(AdaptiveSparkPlanExec.executionContext)
            } catch {
              case e: Throwable =>
                cleanUpAndThrowException(Seq(e), Some(stage.id))
            }
          }
        }

        // Wait on the next completed stage, which indicates new stats are available and probably
        // new stages can be created. There might be other stages that finish at around the same
        // time, so we process those stages too in order to reduce re-planning.
        val nextMsg = events.take()
        val rem = new util.ArrayList[StageMaterializationEvent]()
        events.drainTo(rem)
        (Seq(nextMsg) ++ rem.asScala).foreach {
          case StageSuccess(stage, res) =>
            stage.resultOption.set(Some(res))
          case StageFailure(stage, ex) =>
            errors.append(ex)
        }

        // In case of errors, we cancel all running stages and throw exception.
        if (errors.nonEmpty) {
          cleanUpAndThrowException(errors.toSeq, None)
        }

        // Try re-optimizing and re-planning. Adopt the new plan if its cost is equal to or less
        // than that of the current plan; otherwise keep the current physical plan together with
        // the current logical plan since the physical plan's logical links point to the logical
        // plan it has originated from.
        // Meanwhile, we keep a list of the query stages that have been created since last plan
        // update, which stands for the "semantic gap" between the current logical and physical
        // plans. And each time before re-planning, we replace the corresponding nodes in the
        // current logical plan with logical query stages to make it semantically in sync with
        // the current physical plan. Once a new plan is adopted and both logical and physical
        // plans are updated, we can clear the query stage list because at this point the two plans
        // are semantically and physically in sync again.
        val logicalPlan = replaceWithQueryStagesInLogicalPlan(currentLogicalPlan, stagesToReplace)
        val (newPhysicalPlan, newLogicalPlan) = reOptimize(logicalPlan)
        val origCost = costEvaluator.evaluateCost(currentPhysicalPlan)
        val newCost = costEvaluator.evaluateCost(newPhysicalPlan)
        if (newCost < origCost ||
            (newCost == origCost && currentPhysicalPlan != newPhysicalPlan)) {
          logOnLevel(s"Plan changed from $currentPhysicalPlan to $newPhysicalPlan")
          cleanUpTempTags(newPhysicalPlan)
          currentPhysicalPlan = newPhysicalPlan
          currentLogicalPlan = newLogicalPlan
          stagesToReplace = Seq.empty[QueryStageExec]
        }
        // Now that some stages have finished, we can try creating new stages.
        result = createQueryStages(currentPhysicalPlan)
      }

      // Run the final plan when there's no more unfinished stages.
      currentPhysicalPlan = applyPhysicalRules(
        result.newPlan,
        finalStageOptimizerRules,
        Some((planChangeLogger, "AQE Final Query Stage Optimization")))
      isFinalPlan = true
      executionId.foreach(onUpdatePlan(_, Seq(currentPhysicalPlan)))
      currentPhysicalPlan
    }
  }
```

先判断对于一个AdaptiveSparkPlanExec是否已经是最终的执行计划了，如果是的直接返回，不需要再优化，否则就对`currentPhysicalPlan`执行`createQueryStages`函数，这里`currentPhysicalPlan` 具体如下

```
  @transient private val initialPlan = context.session.withActive {
    applyPhysicalRules(
      inputPlan, queryStagePreparationRules, Some((planChangeLogger, "AQE Preparations")))
  }

  @volatile private var currentPhysicalPlan = initialPlan
```

这里的SparkPlan会应用下`queryStagePreparationRules` 具体规则

```
  private def queryStagePreparationRules: Seq[Rule[SparkPlan]] = Seq(
    RemoveRedundantProjects,
    EnsureRequirements,
    RemoveRedundantSorts,
    DisableUnnecessaryBucketedScan
  ) ++ context.session.sessionState.queryStagePrepRules
```

去除多余的投影、确保输入的数据满足节点Distribution的需要（这个规则里面会增删shuffleExchange）、去除多余的排序、关闭不需要的分桶扫描，应用完这些后就SparkPlan就进入`createQueryStages`

```
  private def createQueryStages(plan: SparkPlan): CreateStageResult = plan match {
    case e: Exchange =>
      // First have a quick check in the `stageCache` without having to traverse down the node.
      context.stageCache.get(e.canonicalized) match {
        case Some(existingStage) if conf.exchangeReuseEnabled =>
          val stage = reuseQueryStage(existingStage, e)
          val isMaterialized = stage.resultOption.get().isDefined
          CreateStageResult(
            newPlan = stage,
            allChildStagesMaterialized = isMaterialized,
            newStages = if (isMaterialized) Seq.empty else Seq(stage))

        case _ =>
          val result = createQueryStages(e.child)
          val newPlan = e.withNewChildren(Seq(result.newPlan)).asInstanceOf[Exchange]
          // Create a query stage only when all the child query stages are ready.
          if (result.allChildStagesMaterialized) {
            var newStage = newQueryStage(newPlan)
            if (conf.exchangeReuseEnabled) {
              // Check the `stageCache` again for reuse. If a match is found, ditch the new stage
              // and reuse the existing stage found in the `stageCache`, otherwise update the
              // `stageCache` with the new stage.
              val queryStage = context.stageCache.getOrElseUpdate(e.canonicalized, newStage)
              if (queryStage.ne(newStage)) {
                newStage = reuseQueryStage(queryStage, e)
              }
            }
            val isMaterialized = newStage.resultOption.get().isDefined
            CreateStageResult(
              newPlan = newStage,
              allChildStagesMaterialized = isMaterialized,
              newStages = if (isMaterialized) Seq.empty else Seq(newStage))
          } else {
            CreateStageResult(newPlan = newPlan,
              allChildStagesMaterialized = false, newStages = result.newStages)
          }
      }

    case q: QueryStageExec =>
      CreateStageResult(newPlan = q,
        allChildStagesMaterialized = q.resultOption.get().isDefined, newStages = Seq.empty)

    case _ =>
      if (plan.children.isEmpty) {
        CreateStageResult(newPlan = plan, allChildStagesMaterialized = true, newStages = Seq.empty)
      } else {
        val results = plan.children.map(createQueryStages)
        CreateStageResult(
          newPlan = plan.withNewChildren(results.map(_.newPlan)),
          allChildStagesMaterialized = results.forall(_.allChildStagesMaterialized),
          newStages = results.flatMap(_.newStages))
      }
  }
```

这里主要对sparkplan分了3种执行逻辑，3种情况中的第一种如果sparkplan 是 `Exchange` 那么就先判断这个sparkplan有没有对应的 QueryStageExec 这里又分了两种，如果有并开启了复用

```
  private def reuseQueryStage(existing: QueryStageExec, exchange: Exchange): QueryStageExec = {
    val queryStage = existing.newReuseInstance(currentStageId, exchange.output)
    currentStageId += 1
    setLogicalLinkForNewQueryStage(queryStage, exchange)
    queryStage
  }
```

currentStageId += 1 并且 `setLogicalLinkForNewQueryStage(queryStage, exchange)` 为物理计划设置逻辑计划的连接, 这时候返回`CreateStageResult`这个只是单纯的包装下`stage`,用来记录子依赖的 `stage`是否执行完，执行完`newStages`就是空，否则就把复用的`stage`作为`newStages`返回; 如果没有的话那就对子sparkPlan递归调用createQueryStages,如果返回的CreateStageResult对象子stages已经完全执行完，那就调用`newQueryStage`用来产生`QueryStageExec`，这里就把`Exchange`切成了子`QueryStageExec`,具体逻辑如下

```
  private def newQueryStage(e: Exchange): QueryStageExec = {
    val optimizedPlan = applyPhysicalRules(
      e.child, queryStageOptimizerRules, Some((planChangeLogger, "AQE Query Stage Optimization")))
    val queryStage = e match {
      case s: ShuffleExchangeLike =>
        val newShuffle = applyPhysicalRules(
          s.withNewChildren(Seq(optimizedPlan)),
          postStageCreationRules,
          Some((planChangeLogger, "AQE Post Stage Creation")))
        if (!newShuffle.isInstanceOf[ShuffleExchangeLike]) {
          throw new IllegalStateException(
            "Custom columnar rules cannot transform shuffle node to something else.")
        }
        ShuffleQueryStageExec(currentStageId, newShuffle)
      case b: BroadcastExchangeLike =>
        val newBroadcast = applyPhysicalRules(
          b.withNewChildren(Seq(optimizedPlan)),
          postStageCreationRules,
          Some((planChangeLogger, "AQE Post Stage Creation")))
        if (!newBroadcast.isInstanceOf[BroadcastExchangeLike]) {
          throw new IllegalStateException(
            "Custom columnar rules cannot transform broadcast node to something else.")
        }
        BroadcastQueryStageExec(currentStageId, newBroadcast)
    }
    currentStageId += 1
    setLogicalLinkForNewQueryStage(queryStage, e)
    queryStage
  }
```

进入方法第一步对Exchange 调用 `queryStageOptimizerRules`

```
  @transient private val queryStageOptimizerRules: Seq[Rule[SparkPlan]] = Seq(
    ReuseAdaptiveSubquery(context.subqueryCache),
    CoalesceShufflePartitions(context.session),
    // The following two rules need to make use of 'CustomShuffleReaderExec.partitionSpecs'
    // added by `CoalesceShufflePartitions`. So they must be executed after it.
    OptimizeSkewedJoin,
    OptimizeLocalShuffleReader
  )
```

这里的几个优化规则就是AQE的核心规则，`ReuseAdaptiveSubquery` 复用AQE 子查询，`CoalesceShufflePartitions` 修正分区数，`OptimizeSkewedJoin` 优化数据倾斜，`OptimizeLocalShuffleReader` 本地shuffle优化，这几个规则后面详细解读，第二步调用`postStageCreationRules` 规则如下

```
  @transient private val postStageCreationRules = Seq(
    ApplyColumnarRulesAndInsertTransitions(context.session.sessionState.columnarRules),
    CollapseCodegenStages()
  )
```

`ApplyColumnarRulesAndInsertTransitions` 行列相关的规则 `CollapseCodegenStages` 代码生成策略， 接下来就是封装`ShuffleQueryStageExec`或`BroadcastQueryStageExec`返回。总结下`newQueryStage`方法就是将`Exchange` 封装成QueryStageExec，因为Exchange已经执行完所以QueryStageExec处理后已经进行完分区数修正和倾斜优化，然后再一次的走一把复用逻辑后返回`CreateStageResult`,如果子stage没有完成这里CreateStageResult 标记为未完成并且把newStages; 3种情况下的第二种如果`sparkPlan` 是 `QueryStageExec` 直接返回 `CreateStageResult`，3种情况下的第三种就是对子节点调用createQueryStages; 对`createQueryStages`总结下，对于`SparkPlan`自低向上返回可以执行的stages,遇到某个节点的所有子stage都执行完成时调用newQueryStage来优化并行度和解决数据倾斜，`CreateStageResult`中newPlan用来表示返回的`sparkPlan`的根、`allChildStagesMaterialized`用来标记子stage是否执行完成，`newStages`用来记录本次可以执行的stages,返回到 `getFinalPhysicalPlan` 方法就是每次获取可以执行的stages然后提交执行，当有stage执行完成会后就更新逻辑计划，并且reOptimize下逻辑计划

```
  private def reOptimize(logicalPlan: LogicalPlan): (SparkPlan, LogicalPlan) = {
    logicalPlan.invalidateStatsCache()
    val optimized = optimizer.execute(logicalPlan)
    val sparkPlan = context.session.sessionState.planner.plan(ReturnAnswer(optimized)).next()
    val newPlan = applyPhysicalRules(
      sparkPlan,
      preprocessingRules ++ queryStagePreparationRules,
      Some((planChangeLogger, "AQE Replanning")))
    (newPlan, optimized)
  }
```

其实就是从新对新的逻辑计划走一遍AE优化策略，然后比较下新的策略是否代价更小或者新的计划有优化，下面接着循环调用createQueryStages，直到子stage全部执行完。