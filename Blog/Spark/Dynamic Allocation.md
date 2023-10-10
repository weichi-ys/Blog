一切的起源`ApplicationMaster`, 启动AM的时候会调用createAllocator, 核心就是创建 `YarnAllocator` 然后开始向yarn申请资源，其中AM的核心代码

```
    allocator = client.createAllocator(
      yarnConf,
      _sparkConf,
      appAttemptId,
      driverUrl,
      driverRef,
      securityMgr,
      localResources)

    allocator.allocateResources()
```

AM启动完成后就调用`allocationThreadImpl`这个方法里面有个死循环周期性的调用`allocator.allocateResources()`，这个方案是用`synchronized`锁住的，所以会有线程等待导致超时的问题

`YarnAllocator` 核心逻辑如下，初始化时获取spark需要的资源量向yarn申请核心逻辑如下

```
  private def initDefaultProfile(): Unit = synchronized {
    allocatedHostToContainersMapPerRPId(DEFAULT_RESOURCE_PROFILE_ID) =
      new HashMap[String, mutable.Set[ContainerId]]()
    runningExecutorsPerResourceProfileId.put(DEFAULT_RESOURCE_PROFILE_ID, mutable.HashSet[String]())
    numExecutorsStartingPerResourceProfileId(DEFAULT_RESOURCE_PROFILE_ID) = new AtomicInteger(0)
    val initTargetExecNum = SchedulerBackendUtils.getInitialTargetExecutorNumber(sparkConf)
    targetNumExecutorsPerResourceProfileId(DEFAULT_RESOURCE_PROFILE_ID) = initTargetExecNum
    val defaultProfile = ResourceProfile.getOrCreateDefaultProfile(sparkConf)
    createYarnResourceForResourceProfile(defaultProfile)
  }

  initDefaultProfile()
```

在确定初始化需要的资源时就会依据是否开启动态资源分配来决定需要的资源数量，相关逻辑代码在`SchedulerBackendUtils`

```
  def getInitialTargetExecutorNumber(
      conf: SparkConf,
      numExecutors: Int = DEFAULT_NUMBER_EXECUTORS): Int = {
    if (Utils.isDynamicAllocationEnabled(conf)) {
      val minNumExecutors = conf.get(DYN_ALLOCATION_MIN_EXECUTORS)
      val initialNumExecutors = Utils.getDynamicAllocationInitialExecutors(conf)
      val maxNumExecutors = conf.get(DYN_ALLOCATION_MAX_EXECUTORS)
      require(initialNumExecutors >= minNumExecutors && initialNumExecutors <= maxNumExecutors,
        s"initial executor number $initialNumExecutors must between min executor number " +
          s"$minNumExecutors and max executor number $maxNumExecutors")

      initialNumExecutors
    } else {
      conf.get(EXECUTOR_INSTANCES).getOrElse(numExecutors)
    }
  }
```

可以看到如果没有开启动态资源就直接用`spark.executor.instances` 配置的实例数，否则使用 `Utils.getDynamicAllocationInitialExecutors(conf)`的返回值，该值就是`spark.dynamicAllocation.minExecutors` 、 `spark.dynamicAllocation.initialExecutors` 、 `spark.executor.instances` 中的最大值

```
val initialExecutors = Seq(
      conf.get(DYN_ALLOCATION_MIN_EXECUTORS),
      conf.get(DYN_ALLOCATION_INITIAL_EXECUTORS),
      conf.get(EXECUTOR_INSTANCES).getOrElse(0)).max
```

资源初始化完成，如果是静态资源分配这样就结束了, 动态资源分配则还需要依据需要的资源进行伸缩，核心类`ExecutorAllocationManager` 在sparkContext初始化的时候进行构造

```
    _executorAllocationManager =
      if (dynamicAllocationEnabled) {
        schedulerBackend match {
          case b: ExecutorAllocationClient =>
            Some(new ExecutorAllocationManager(
              schedulerBackend.asInstanceOf[ExecutorAllocationClient], listenerBus, _conf,
              cleaner = cleaner, resourceProfileManager = resourceProfileManager))
          case _ =>
            None
        }
      } else {
        None
      }
    _executorAllocationManager.foreach(_.start())
```

我们可以看下`start`方法主要就是将`ExecutorAllocationListener` 加入到`listenerBus` 里，其次就是调用下`client.requestTotalExecutors`，最终由`CoarseGrainedSchedulerBackend`调用`requestTotalExecutors`完成资源的调用，我们来看下核心逻辑

```
    val response = synchronized {
      this.requestedTotalExecutorsPerResourceProfile.clear()
      this.requestedTotalExecutorsPerResourceProfile ++= resourceProfileToNumExecutors
      this.numLocalityAwareTasksPerResourceProfileId = numLocalityAwareTasksPerResourceProfileId
      this.rpHostToLocalTaskCount = hostToLocalTaskCount
      doRequestTotalExecutors(requestedTotalExecutorsPerResourceProfile.toMap)
    }
    defaultAskTimeout.awaitResult(response)
```

这里其实就是发送rpc到am，一般情况下这里可以迅速返回但是如果am那边竞争锁这里就可能超时，这就是为什么会有日志返回一堆节点的原因。这里在额外补充一个知识点，调度是什么时候开始的呢，核心逻辑就在`CoarseGrainedSchedulerBackend`.`isReady` 方法，这里主要涉及下面这些配置`spark.scheduler.maxRegisteredResourcesWaitingTime` 定义了最大的等待时间，`spark.scheduler.minRegisteredResourcesRatio`定义了已有的节点/初始化节点的占比。下面我们主要看`ExecutorAllocationManager`这个类，架构上面其实就是一个根据消息总线调用`ExecutorAllocationClient`进行资源伸缩的设计