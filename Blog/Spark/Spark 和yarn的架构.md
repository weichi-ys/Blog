## 目录

1.  [Hadoop YARN 架构](https://matnoble.github.io/tech/spark/spark-on-yarn/#hadoop-yarn-%E6%9E%B6%E6%9E%84-1)
    1.  [资源管理](https://matnoble.github.io/tech/spark/spark-on-yarn/#%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86)
    2.  [计算调度](https://matnoble.github.io/tech/spark/spark-on-yarn/#%E8%AE%A1%E7%AE%97%E8%B0%83%E5%BA%A6)
2.  [Spark 架构](https://matnoble.github.io/tech/spark/spark-on-yarn/#spark-%E6%9E%B6%E6%9E%84)
3.  [Spark on YARN 架构](https://matnoble.github.io/tech/spark/spark-on-yarn/#spark-on-yarn-%E6%9E%B6%E6%9E%84)
4.  [总结](https://matnoble.github.io/tech/spark/spark-on-yarn/#%E6%80%BB%E7%BB%93)

**摘要**：本文介绍了 Spark on YARN 架构 的基本原理和特点。文章首先介绍了 Hadoop YARN 架构 的两个核心组件：资源管理和计算调度，然后介绍了 Spark 架构 的主要组成部分和功能。文章最后总结了 Spark on YARN 架构 的优势和应用场景。

## [](https://matnoble.github.io/tech/spark/spark-on-yarn/#hadoop-yarn-%E6%9E%B6%E6%9E%84-1)[Hadoop YARN 架构](https://matnoble.github.io/tech/spark/spark-on-yarn/#contents:hadoop-yarn-%E6%9E%B6%E6%9E%84-1) [\[1\]](https://matnoble.github.io/tech/spark/spark-on-yarn/#fn:1)

YARN 本身是一个资源调度框架，负责对运行在内部的计算框架进行资源调度管理。其为分布式计算任务提供两个_独立_的进程：

-   资源管理
-   计算调度

### [](https://matnoble.github.io/tech/spark/spark-on-yarn/#%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86)[资源管理](https://matnoble.github.io/tech/spark/spark-on-yarn/#contents:%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86)

 ![资源层面](https://cdn.jsdelivr.net/gh/MatNoble/Images/win/yarn1-202309012128542.png "资源层面") ◎ 资源层面

在集群中，管理资源有两个角色，组成数据计算框架

-   `ResourceManager`  
    负责为各任务分配资源
-   `NodeManager`  
    每个计算节点上仅且仅有 1 个 `NodeManager`，负责向 `ResourceManager` “上报” 节点信息（CPU，内存，磁盘，网络等监控信息）

### [](https://matnoble.github.io/tech/spark/spark-on-yarn/#%E8%AE%A1%E7%AE%97%E8%B0%83%E5%BA%A6)[计算调度](https://matnoble.github.io/tech/spark/spark-on-yarn/#contents:%E8%AE%A1%E7%AE%97%E8%B0%83%E5%BA%A6)

 ![计算层面](https://cdn.jsdelivr.net/gh/MatNoble/Images/win/yarn3-202309012130754.png "计算层面") ◎ 计算层面

_计算_发生在 `NodeManager` 所在的节点上，有两个角色：

-   `ApplicationMaster` 负责向 `ResourcesManager` 申请资源，并调度作业（Task）
-   `Container` 负责执行具体的计算作业

在 YARN 中，每个应用程序都有一个 ApplicationMaster ，负责与 ResourceManager 交互申请资源，并与 NodeManager 交互启动和监控任务。

## [](https://matnoble.github.io/tech/spark/spark-on-yarn/#spark-%E6%9E%B6%E6%9E%84)[Spark 架构](https://matnoble.github.io/tech/spark/spark-on-yarn/#contents:spark-%E6%9E%B6%E6%9E%84)

 ![Spark 架构](https://spark.apache.org/docs/latest/img/cluster-overview.png "Spark 架构") ◎ Spark 架构

Spark 与 YARN 非常类似，都有资源管理模块和计算模块，这两个模块都有各自的管理进程和执行进程。

Spark 架构 由以下几个部分组成：

-   Driver 程序：负责创建 SparkContext 对象，并将应用程序逻辑划分为多个 Task
-   Executor 进程：负责执行 Task，并将结果返回给 Driver
-   Cluster Manager ：负责管理集群中的资源和节点，可以是 YARN ，也可以是其他类型
-   SparkContext 对象：负责与 Cluster Manager 交互申请资源，并创建 Executor
-   RDD ：弹性分布式数据集，是 Spark 的核心抽象，表示一个不可变、分区、可并行操作的数据集合
-   DAGScheduler ：负责将 RDD 的操作转换为 DAG （有向无环图），并将 DAG 划分为多个 Stage
-   TaskScheduler ：负责将 Stage 划分为多个 Task，并将 Task 分配给 Executor 执行

## [](https://matnoble.github.io/tech/spark/spark-on-yarn/#spark-on-yarn-%E6%9E%B6%E6%9E%84)[Spark on YARN 架构](https://matnoble.github.io/tech/spark/spark-on-yarn/#contents:spark-on-yarn-%E6%9E%B6%E6%9E%84)

Spark on YARN 是指将 Spark 应用程序运行在 YARN 集群上，利用 YARN 提供的资源管理和计算调度功能。在这种架构下，Spark 中的角色和 YARN 中的角色一一对应：

-   `ClusterManager` 相当于 `ResourceManager`，负责资源调度
-   `WorkerNode` 相当于 `NodeManager`，负责管理节点状态
-   `Driver` 相当于 `ApplicationMaster`，负责调度作业，可以运行在集群外部的客户端，也可以运行在集群内部的某个节点上
-   `Executor` 相当于 `Container`，负责具体的计算任务

> Spark on YARN 架构的优势：

-   可以充分利用 YARN 集群的资源，实现资源的动态分配和共享，避免资源的浪费和冲突
-   可以与其他运行在 YARN 上的应用程序协同工作，实现数据的高效处理和分析
-   可以享受 YARN 提供的高可用性、安全性、监控和管理等功能，提高 Spark 应用程序的稳定性和可靠性
-   可以灵活地选择 Spark 应用程序的运行模式，根据不同的场景和需求进行优化和调整

> Spark on YARN 架构的应用场景

-   需要处理大规模、多样化、复杂的数据，例如机器学习、数据挖掘、图计算等
-   需要与其他运行在 YARN 上的应用程序进行数据交互，例如 Hadoop MapReduce、Hive 等
-   需要在同一个集群上运行多个 Spark 应用程序，实现资源的合理分配和利用
-   需要保证 Spark 应用程序的高可用性、安全性、监控和管理等功能

## [](https://matnoble.github.io/tech/spark/spark-on-yarn/#%E6%80%BB%E7%BB%93)[总结](https://matnoble.github.io/tech/spark/spark-on-yarn/#contents:%E6%80%BB%E7%BB%93)

本文介绍了 Spark on YARN 架构 的基本原理和特点，以及它的优势和应用场景。Spark on YARN 架构 是一种将 Spark 应用程序运行在 YARN 集群上的架构，可以充分利用 YARN 提供的资源管理和计算调度功能，实现数据的高效处理和分析。Spark on YARN 架构适合处理大规模、多样化、复杂的数据，与其他运行在 YARN 上的应用程序协同工作，以及在同一个集群上运行多个 Spark 应用程序。