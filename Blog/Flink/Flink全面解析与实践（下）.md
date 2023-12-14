本文已收录至GitHub，推荐阅读 👉 [Java随想录](https://github.com/ZhengShuHai/JavaRecord)  
微信公众号：Java随想录

> 原创不易，注重版权。转载请注明原作者和原文链接

**承接上篇未完待续的话题，我们一起继续Flink的深入探讨**

## Flink State状态

Flink是一个有状态的流式计算引擎，所以会将中间计算结果（状态）进行保存，默认保存到TaskManager的堆内存中。

但是当Task挂掉，那么这个Task所对应的状态都会被清空，造成了数据丢失，无法保证结果的正确性，哪怕想要得到正确结果，所有数据都要重新计算一遍，效率很低。

想要保证 **At -least-once** 和 **Exactly-once**，则需要把数据状态持久化到更安全的存储介质中，Flink提供了堆内内存、堆外内存、HDFS、RocksDB等存储介质。

先来看下Flink提供的状态有哪些，Flink中状态可以分为两种类型：

-   **Keyed State**
    
    基于KeyedStream上的状态，这个状态是跟特定的Key绑定，KeyedStream流上的每一个Key都对应一个State，每一个Operator可以启动多个Thread处理，但是相同Key的数据只能由同一个Thread处理，因此一个Keyed状态只能存在于某一个Thread中，一个Thread会有多个Keyed State。
    
-   **Non-Keyed State（Operator State）**
    
    Operator State与Key无关，而是与Operator绑定，整个Operator只对应一个State。比如：Flink中的Kafka Connector就使用了Operator State，它会在每个Connector实例中，保存该实例消费Topic的所有（partition, offset）映射。
    

Flink针对Keyed State提供了以下可以保存State的数据结构：

-   **ValueState**：类型为T的单值状态，这个状态与对应的Key绑定，最简单的状态，通过update更新值，通过value获取状态值。
-   **ListState**：Key上的状态值为一个列表，这个列表可以通过`add()`方法往列表中添加值，也可以通过`get()`方法返回一个Iterable来遍历状态值。
-   **ReducingState**：每次调用`add()`方法添加值的时候，会调用用户传入的`reduceFunction`，最后合并到一个单一的状态值。
-   **MapState**：状态值为一个Map，用户通过`put()`或`putAll()`方法添加元素，get(key)通过指定的key获取value，使用`entries()`、`keys()`、`values()`检索。
-   **AggregatingState**：保留一个单值，表示添加到状态的所有值的聚合。和 `ReducingState` 相反的是, 聚合类型可能与添加到状态的元素的类型不同。使用 `add(IN)` 添加的元素会调用用户指定的 `AggregateFunction` 进行聚合。
-   **FoldingState**：已过时，建议使用AggregatingState 保留一个单值，表示添加到状态的所有值的聚合。 与 `ReducingState` 相反，聚合类型可能与添加到状态的元素类型不同。 使用`add（T）`添加的元素会调用用户指定的 `FoldFunction` 折叠成聚合值。

**案例1：使用ValueState统计每个键的当前计数**

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.fromElements(Tuple2.of("user1", 1), Tuple2.of("user2", 1), Tuple2.of("user1", 1), Tuple2.of("user2", 1)) .keyBy(0) .flatMap(new CountWithKeyedState()) .print(); env.execute("Flink ValueState example"); } public static class CountWithKeyedState extends RichFlatMapFunction<Tuple2<String, Integer>, Tuple2<String, Integer>> { private transient ValueState<Integer> countState; @Override public void open(Configuration parameters) throws Exception { ValueStateDescriptor<Integer> descriptor = new ValueStateDescriptor<>("countState", Integer.class, 0); countState = getRuntimeContext().getState(descriptor); } @Override public void flatMap(Tuple2<String, Integer> value, Collector<Tuple2<String, Integer>> out) throws Exception { Integer currentCount = countState.value(); currentCount += value.f1; countState.update(currentCount); out.collect(Tuple2.of(value.f0, currentCount)); } }
```

在这段代码中，我们首先创建了一个 `StreamExecutionEnvironment`，然后产生一些元素，每个元素都是指定用户的一个事件。`keyBy(0)` 表示我们以元组的第一个字段（即用户ID）为键进行分组。

然后，我们使用 `flatMap` 算子应用了 `CountWithKeyedState` 函数。这个函数使用了 Flink 的 ValueState 来存储和更新每个键的当前计数。

在 `open` 方法中，我们定义了一个名为 "countState" 的 ValueState，并把它初始化为 0。在 `flatMap` 方法中，我们从 ValueState 中获取当前计数，增加输入元素的值，然后更新 ValueState，并发出带有当前总数的元组。

注意：在真实的生产环境中，你可能需要从数据源（如 Kafka, HDFS等）读取数据，而不是使用 `fromElements` 方法

**案例2：使用MapState 统计单词出现次数**

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.socketTextStream("localhost", 9999) .flatMap(new Tokenizer()) .keyBy(value -> value.f0) .flatMap(new RichFlatMapFunction<Tuple2<String, Integer>, Tuple2<String, Integer>>() { private transient MapState<String, Integer> wordState; @Override public void open(Configuration parameters){ MapStateDescriptor<String, Integer> descriptor = new MapStateDescriptor<>("wordCount", String.class, Integer.class); wordState = getRuntimeContext().getMapState(descriptor); } @Override public void flatMap(Tuple2<String, Integer> value, Collector<Tuple2<String, Integer>> out) throws Exception { Integer count = wordState.get(value.f0); if (count == null) { count = 0; } count += value.f1; wordState.put(value.f0, count); out.collect(Tuple2.of(value.f0, count)); } }) .print(); env.execute("Word Count with MapState"); } public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> { @Override public void flatMap(String value, Collector<Tuple2<String, Integer>> out) { String[] words = value.toLowerCase().split("\\W+"); for (String word : words) { if (word.length() > 0) { out.collect(new Tuple2<>(word, 1)); } } } }
```

在这个例子中，我们首先通过 `socketTextStream` 方法从本地的 socket 获取输入数据流。然后我们用 `flatMap` 操作将每行输入分解为单个单词，并且为每个单词赋予基础计数值（基数）1。

我们创建一个使用 `RichFlatMapFunction` 的 operator，它可以访问 `MapState`。在 `open()` 方法中，我们定义了 `MapStateDescriptor`，然后用这个 `descriptor` 创建 `MapState`。

在 `flatMap()` 函数中，我们获取当前单词的计数值，如果不存在则设置为0。然后我们增加计数值，更新 MapState，并且输出当前单词和它的出现次数。

**案例3：使用ReducingState统计输入流中每个键的最大值**

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<Tuple2<String, Integer>> dataStream = env.fromElements( Tuple2.of("A", 6), Tuple2.of("B", 5), Tuple2.of("C", 4), Tuple2.of("A", 3), Tuple2.of("B", 2), Tuple2.of("C", 1) ); dataStream.keyBy(0).flatMap(new MaxValueReducer()).print(); env.execute("ReducingState Example"); } public static class MaxValueReducer extends RichFlatMapFunction<Tuple2<String, Integer>, Tuple2<String, Integer>> { private transient ReducingState<Integer> maxState; @Override public void open(Configuration config) { ReducingStateDescriptor<Integer> descriptor = new ReducingStateDescriptor<>( "maxValue", // state的名字 Math::max, // ReduceFunction，这里取两者的最大值 TypeInformation.of(Integer.class)); // 类型信息 maxState = getRuntimeContext().getReducingState(descriptor); } @Override public void flatMap(Tuple2<String, Integer> input, Collector<Tuple2<String, Integer>> out) throws Exception { maxState.add(input.f1); // 更新state的值 out.collect(Tuple2.of(input.f0, maxState.get())); // 输出当前key的最大值 } }
```

在上述代码中，我们首先创建了一个新的`MaxValueReducer`类，该类扩展了`RichFlatMapFunction`。然后定义了一个`ReducingState`变量，用于在每个key上维护最大值。在`open()`方法中，我们初始化了这个状态变量。在`flatMap()`方法中，我们简单地将新的值添加到状态中，并输出当前key的最大值。

输出如下：

```scss
7> (A,6) 7> (A,6) 2> (B,5) 2> (C,4) 2> (B,5) 2> (C,4)
```

**案例4：使用AggregatingState统计输入流中每个键的平均值**

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<Tuple2<String, Integer>> input = env.fromElements( Tuple2.of("A", 6), Tuple2.of("B", 5), Tuple2.of("C", 4), Tuple2.of("A", 3), Tuple2.of("B", 2), Tuple2.of("C", 1) ); input.keyBy(x -> x.f0) .process(new AggregatingProcessFunction()) .print(); env.execute(); } public static class AverageAggregate implements AggregateFunction<Integer, Tuple2<Integer, Integer>, Double> { @Override public Tuple2<Integer, Integer> createAccumulator() { return new Tuple2<>(0, 0); } @Override public Tuple2<Integer, Integer> add(Integer value, Tuple2<Integer, Integer> accumulator) { return new Tuple2<>(accumulator.f0 + value, accumulator.f1 + 1); } @Override public Double getResult(Tuple2<Integer, Integer> accumulator) { return ((double) accumulator.f0) / accumulator.f1; } @Override public Tuple2<Integer, Integer> merge(Tuple2<Integer, Integer> a, Tuple2<Integer, Integer> b) { return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1); } } public static class AggregatingProcessFunction extends KeyedProcessFunction<String, Tuple2<String, Integer>, Tuple2<String, Double>> { private AggregatingState<Integer, Double> avgState; @Override public void open(Configuration parameters) { AggregatingStateDescriptor<Integer, Tuple2<Integer, Integer>, Double> descriptor = new AggregatingStateDescriptor<>("average", new AverageAggregate(), TypeInformation.of(new TypeHint<Tuple2<Integer, Integer>>() { })); avgState = getRuntimeContext().getAggregatingState(descriptor); } @Override public void processElement(Tuple2<String, Integer> value, Context ctx, Collector<Tuple2<String, Double>> out) throws Exception { avgState.add(value.f1); out.collect(new Tuple2<>(value.f0, avgState.get())); } }
```

输入如下：

```scss
7> (A,6.0) 2> (B,5.0) 2> (C,4.0) 7> (A,4.5) 2> (B,3.5) 2> (C,2.5)
```

这段代码主要是计算每个键对应的值的平均数。代码中定义了：`AverageAggregate`和`AggregatingProcessFunction`。

`AverageAggregate`类实现了`AggregateFunction`接口，用于计算平均值：

-   `createAccumulator`方法返回一个新的累加器，这里是一个包含两个整数的元组，表示当前的总数和元素的数量。
-   `add`方法向累加器添加一个元素的值，将其添加到总数中，并增加元素数量。
-   `getResult`方法根据累加器计算平均值。
-   `merge`方法合并两个累加器，将他们的总数和元素数量相加。

`AggregatingProcessFunction`类扩展了`KeyedProcessFunction`，在接收到一个元素时添加到状态中的平均值，并输出当前的平均值：

-   在`open`方法中，创建了一个`AggregatingStateDescriptor`，描述要保存的状态，这里保存的是平均值。
-   `processElement`方法在接收到一个新元素时，将其值添加到状态中的平均值，然后输出包含键和当前平均值的元组。

以上案例代码都经过本地运行和测试，建议大家自行运行以便更深入地理解。

### CheckPoint & SavePoint

**有状态流应用中的检查点（CheckPoint），其实就是所有任务的状态在某个时间点的一个快照（一份拷贝）**

简单来讲，就是一次「**存盘**」，让我们之前处理数据的进度不要丢掉。在一个流应用程序运行时，Flink 会定期保存检查点，在检查点中会记录每个算子的 id 和状态。

如果发生故障，Flink 就会用最近一次成功保存的检查点来恢复应用的状态，重新启动处理流程，就如同「**读档**」一样。

默认情况下，检查点是被禁用的，需要在代码中手动开启。直接调用执行环境的`enableCheckpointing()`方法就可以开启检查点。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.enableCheckpointing(1000);
```

这里传入的参数是检查点的间隔时间，单位为毫秒。

除了检查点之外，Flink 还提供了「**保存点（SavePoint）**」的功能。

保存点在原理和形式上跟检查点完全一样，也是状态持久化保存的一个快照。

保存点与检查点最大的区别，就是触发的时机。检查点是由 Flink 自动管理的，定期创建，发生故障之后自动读取进行恢复，这是一个「**自动存盘**」的功能。而保存点不会自动创建，必须由用户明确地手动触发保存操作，所以就是「**手动存盘**」。

因此两者尽管原理一致，但用途就有所差别了。

**检查点主要用来做故障恢复，是容错机制的核心；保存点则更加灵活，可以用来做有计划的手动备份和恢复**

检查点具体的持久化存储位置，取决于「**检查点存储（CheckPointStorage）**」的设置。

默认情况下，检查点存储在 JobManager 的堆（heap）内存中。而对于大状态的持久化保存，Flink也提供了在其他存储位置进行保存的接口，这就是「 **CheckPointStorage**」。

具体可以通过调用检查点配置的 `setCheckpointStorage()`来配置，需要传入一个CheckPointStorage 的实现类。例如：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); // 设置检查点时间间隔为1000ms env.enableCheckpointing(1000); // 设置checkpoint存储路径, 注意路径需要是可访问且有写权限的HDFS或本地路径 URI checkpointPath = URI.create("hdfs://localhost:9000/flink-checkpoints"); FileSystemCheckpointStorage storage = new FileSystemCheckpointStorage(checkpointPath, 10000); // 应用配置 env.getCheckpointConfig().setCheckpointStorage(storage); // 设置重启策略，这里我们设置为固定延时无限重启 //Flink的重启策略是用来决定如何处理作业执行过程中出现的失败情况的。如果Flink作业在运行时出错，比如由于代码错误、硬件故障或 网络问题等，那么重启策略就会决定是否和如何重启作业。 env.setRestartStrategy(RestartStrategies.fixedDelayRestart( // 尝试重启次数 3, //每次尝试重启的固定延迟时间为 10 秒 org.apache.flink.api.common.time.Time.of(10, java.util.concurrent.TimeUnit.SECONDS) )); env.execute("Flink Checkpoint Example"); }
```

Flink 主要提供了两种 CheckPointStorage：

-   作业管理器的堆内存（JobManagerCheckpointStorage）
-   文件系统（FileSystemCheckpointStorage）

对于实际生产应用，我们一般会将 CheckPointStorage 配置为高可用的分布式文件系统（HDFS，S3 等）。

#### CheckPoint原理

Flink会在输入的数据集上间隔性地生成**CheckPoint Barrier**，通过栅栏（Barrier）将间隔时间段内的数据划分到相应的CheckPoint中。

当程序出现异常时，Operator就能够从上一次快照中恢复所有算子之前的状态，从而保证数据的一致性。

例如在Kafka Consumer算子中维护offset状态，当系统出现问题无法从Kafka中消费数据时，可以将offset记录在状态中，当任务重新恢复时就能够从指定的偏移量开始消费数据。

默认情况Flink不开启检查点，用户需要在程序中通过调用方法配置来开启检查点，另外还可以调整其他相关参数

-   CheckPoint 开启和时间间隔指定
    
    开启检查点并且指定检查点时间间隔为1000ms，根据实际情况自行选择，如果状态比较大，则建议适当增加该值
    
    ```java
    env.enableCheckpointing(1000)
    ```
    
-   Exactly-once 和 At-least-once语义选择
    
    选择Exactly-once语义保证整个应用内端到端的数据一致性，这种情况比较适合于数据要求比较高，不允许出现丢数据或者数据重复，与此同时，Flink的性能也相对较弱。
    
    而At-least-once语义更适合于时廷和吞吐量要求非常高但对数据的一致性要求不高的场景。如下通过`setCheckpointingMode()`方法来设定语义模式，**默认情况下使用的是Exactly-once模式**。
    
    ```java
    env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
    ```
    
-   CheckPoint 超时时间
    
    超时时间指定了每次CheckPoint执行过程中的上限时间范围，一旦CheckPoint执行时间超过该阈值，Flink将会中断CheckPoint过程，并按照超时处理。该指标可以通过`setCheckpointTimeout()`方法设定，默认为10分钟
    
    ```java
    env.getCheckpointConfig().setCheckpointTimeout(5 * 60 * 1000);
    ```
    
-   CheckPoint 最小时间间隔
    
    该参数主要目的是设定两个CheckPoint之间的最小时间间隔，防止Flink应用密集地触发CheckPoint操作，会占用了大量计算资源而影响到整个应用的性能
    
    ```java
    env.getCheckpointConfig().setMinPauseBetweenCheckpoints(600)
    ```
    
-   CheckPoint 最大并行执行数量
    
    在默认情况下只有一个检查点可以运行，根据用户指定的数量可以同时触发多个CheckPoint，进而提升CheckPoint整体的效率
    
    ```java
    env.getCheckpointConfig().setMaxConcurrentCheckpoints(1)
    ```
    
-   任务取消后，是否删除 CheckPoint 中保存的数据
    
    `RETAIN_ON_CANCELLATION`：表示一旦Flink处理程序被cancel后，会保留CheckPoint数据，以便根据实际需要恢复到指定的CheckPoint。
    
    `DELETE_ON_CANCELLATION`：表示一旦Flink处理程序被cancel后，会删除CheckPoint数据，只有Job执行失败的时候才会保存CheckPoint。
    
    ```java
    env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION)
    ```
    
-   容忍的检查的失败数
    
    设置可以容忍的检查的失败数，超过这个数量则系统自动关闭和停止任务。
    
    ```java
    env.getCheckpointConfig().setTolerableCheckpointFailureNumber(1)
    ```
    

#### SavePoint原理

SavePoint 底层实现其实也是使用CheckPoint的机制。

SavePoint是用户以手工命令的方式触发Checkpoint，并将结果持久化到指定的存储路径中，其主要目的是帮助用户在升级和维护集群过程中保存系统中的状态数据，避免因为停机运维或者升级应用等正常终止应用的操作而导致系统无法恢复到原有的计算状态的情况，从而无法实现从端到端的 Excatly-Once 语义保证。

要使用SavePoint，需要按照以下步骤进行：

1.  **配置状态后端**： 在Flink中，状态可以保存在不同的后端存储中，例如内存、文件系统或分布式存储系统（如HDFS）。要启用SavePoint，需要在Flink配置文件中配置合适的状态后端。
    
    通常，使用分布式存储系统作为状态后端是比较常见的做法，因为它可以提供更好的可靠性和容错性。
    
2.  **生成SavePoint**： 在Flink应用程序运行时，可以通过以下方式手动触发生成SavePoint：
    
    ```bash
    bin/flink savepoint <jobID> [targetDirectory]
    ```
    
    其中，`<jobID>`是要保存状态的Flink作业的Job ID，`[targetDirectory]`是可选的目标目录，用于保存SavePoint数据。如果没有提供`targetDirectory`，SavePoint将会保存到Flink配置中所配置的状态后端中。
    
3.  **恢复SavePoint**： 要恢复到SavePoint状态，可以通过以下方式提交作业：
    
    ```bash
    bin/flink run -s :savepointPath [:runArgs]
    ```
    
    其中，`savepointPath`是之前生成的SavePoint的路径，`runArgs`是提交作业时的其他参数。
    
4.  **确保应用程序状态的兼容性**： 在使用SavePoint时，应用程序的状态结构和代码必须与生成SavePoint的版本保持兼容。这意味着在更新应用程序代码后，可能需要做一些额外的工作来保证状态的向后兼容性，以便能够成功恢复到旧的SavePoint。
    

### StateBackend状态后端

在Flink中提供了StateBackend来存储和管理状态数据。

Flink一共实现了三种类型的状态管理器：`MemoryStateBackend`、`FsStateBackend`、`RocksDBStateBackend`。

#### MemoryStateBackend

基于内存的状态管理器，将状态数据全部存储在JVM堆内存中。

基于内存的状态管理具有非常快速和高效的特点，但也具有非常多的限制，最主要的就是内存的容量限制，一旦存储的状态数据过多就会导致系统内存溢出等问题，从而影响整个应用的正常运行。

同时如果机器出现问题，整个主机内存中的状态数据都会丢失，进而无法恢复任务中的状态数据。因此从数据安全的角度建议用户尽可能地避免在生产环境中使用MemoryStateBackend。

**MemoryStateBackend是Flink的默认状态后端管理器**

```java
env.setStateBackend(new MemoryStateBackend(100*1024*1024));
```

注意：聚合类算子的状态会同步到 JobManager 内存中，因此对于聚合类算子比较多的应用会对 JobManager 的内存造成一定的压力，进而影响集群。

#### FsStateBackend

和MemoryStateBackend有所不同的是，FsStateBackend是基于文件系统的一种状态管理器，这里的文件系统可以是本地文件系统，也可以是HDFS分布式文件系统。

```java
env.setStateBackend(new FsStateBackend("path",true));
```

如果path是本地文件路径，格式为：`file:///`；如果path是HDFS文件路径，格式为：`hdfs://`。

第二个参数代表是否异步保存状态数据到HDFS，异步方式能够尽可能避免ChecPoint的过程中影响流式计算任务。

FsStateBackend更适合任务量比较大的应用，例如：包含了时间范围非常长的窗口计算，或者状态比较大的场景。

#### RocksDBStateBackend

RocksDBStateBackend是Flink中内置的第三方状态管理器，和前面的状态管理器不同，RocksDBStateBackend需要单独引入相关的依赖包到工程中。

```xml
<dependency> <groupId>org.apache.flink</groupId> <artifactId>flink-statebackend-rocksdb_2.12</artifactId> <version>1.14.4</version> <scope>test</scope> </dependency>
```

```java
env.setStateBackend(new RocksDBStateBackend("file:///tmp/flink-backend"));
```

RocksDBStateBackend采用异步的方式进行状态数据的Snapshot，任务中的状态数据首先被写入本地RockDB中，这样在RockDB仅会存储正在进行计算的热数据，而需要进行CheckPoint的时候，会把本地的数据直接复制到远端的FileSystem中。

与FsStateBackend相比，RocksDBStateBackend在性能上要比FsStateBackend高一些，主要是因为借助于RocksDB在本地存储了最新热数据，然后通过异步的方式再同步到文件系统中，但RocksDBStateBackend和MemoryStateBackend相比性能就会较弱一些。

RocksDB克服了State受内存限制的缺点，同时又能够持久化到远端文件系统中，推荐在生产中使用。

#### 集群级配置StateBackend

全局配置需要修改集群中的配置文件`flink-conf.yaml`。

-   配置FsStateBackend

```yaml
state.backend: filesystem state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints
```

-   配置MemoryStateBackend

```yaml
state.backend: jobmanager
```

-   配置RocksDBStateBackend

```yaml
#同时操作RocksDB的线程数 state.backend.rocksdb.checkpoint.transfer.thread.num: 1 #RocksDB存储状态数据的本地文件路径 state.backend.rocksdb.localdir: 本地path
```

## Window

在流处理中，我们往往需要面对的是连续不断、无休无止的无界流，不可能等到所有数据都到齐了才开始处理。

所以聚合计算其实在实际应用中，我们往往更关心一段时间内数据的统计结果，比如在过去的 1 分钟内有多少用户点击了网页。在这种情况下，我们就可以定义一个窗口，收集最近一分钟内的所有用户点击数据，然后进行聚合统计，最终输出一个结果就可以了。

**窗口实质上是将无界流切割为一系列有界流，采用左开右闭的原则**

**Flink中的窗口分为两类：基于时间的窗口（Time-based Window）和基于数量的窗口（Count-based Window）**

-   时间窗口（Time Window）：按照时间段去截取数据，这在实际应用中最常见。
-   计数窗口（Count Window）：由数据驱动，也就是说按照固定的个数，来截取一段数据集。

时间窗口中又包含了：**滚动时间窗口、滑动时间窗口、会话窗口**

计数窗口包含了：**滚动计数窗口、滑动计数窗口**

时间窗口、计数窗口只是对窗口的一个大致划分。在具体应用时，还需要定义更加精细的规则，来控制数据应该划分到哪个窗口中去。不同的分配数据的方式，就可以有不同的功能应用。

根据分配数据的规则，窗口的具体实现可以分为 4 类：**滚动窗口（Tumbling Window）、滑动窗口（Sliding Window）、会话窗口（Session Window）、全局窗口（Global Window）**

### 滚动窗口

**滚动窗口每个窗口的大小固定，且相邻两个窗口之间没有重叠**

滚动窗口可以基于时间定义，也可以基于数据个数定义，需要的参数只有窗口大小。

我们可以定义一个大小为1小时的滚动时间窗口，那么每个小时就会进行一次统计；或者定义一个大小为10的滚动计数窗口，就会每10个数进行一次统计。

基于时间的滚动窗口：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<Tuple2<Integer, Integer>> randomKeyedStream = env .fromSequence(1, Long.MAX_VALUE) // 将每个数映射为一个二元组，第一个元素是随机键，第二个元素是数本身 .map(new MapFunction<Long, Tuple2<Integer, Integer>>() { private final Random rnd = new Random(); @Override public Tuple2<Integer, Integer> map(Long value) { return new Tuple2<>(rnd.nextInt(10), value.intValue()); } }); // 对流进行滚动窗口操作，窗口大小为5秒 // 应用窗口函数，求每个窗口的和 DataStream<Integer> sum = randomKeyedStream .assignTimestampsAndWatermarks(WatermarkStrategy .<Tuple2<Integer, Integer>>forBoundedOutOfOrderness(Duration.ofSeconds(5)) .withTimestampAssigner((event, timestamp) -> event.f1)) .keyBy(0) .timeWindow(Time.seconds(5)) .apply(new WindowFunction<Tuple2<Integer, Integer>, Integer, Tuple, TimeWindow>() { @Override public void apply(Tuple key, TimeWindow window, Iterable<Tuple2<Integer, Integer>> values, Collector<Integer> out){ int sum1 = 0; for (Tuple2<Integer, Integer> val: values) { sum1 += val.f1; } out.collect(sum1); } }); sum.print(); env.execute("Tumbling Window Example"); }
```

这个程序的主要功能是从1到`Long.MAX_VALUE`产生一个序列，并为每个生成的数字创建一个二元组（Tuple2），然后在5秒大小的窗口上对二元组进行操作并输出每个窗口中所有值的总和。

详细解释如下：

1.  `StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();`: 获取Flink的运行环境。
2.  产生一个无限长的序列（从1开始到最大的Long型数），每个数字都映射成一个二元组，第一个元素（f0）是一个0-9的随机整数（作为键用于之后的keyBy操作），第二个元素（f1）是数字本身。
3.  使用`assignTimestampsAndWatermarks`来定义事件时间和水位线。这里设定了最大延迟时间为5秒(`forBoundedOutOfOrderness`)，并将二元组的第二个元素作为时间戳。
4.  使用`keyBy(0)`按照二元组的第一个元素进行分区，这样保证了相同键的元素会被发送到同一个任务中。
5.  定义了一个5秒的滚动窗口`timeWindow(Time.seconds(5))`。
6.  使用`apply`函数应用在每个窗口上，计算每个窗口中所有二元组的第二个元素（f1）的总和，并收集结果。最终，每个窗口计算的总和都会被输出。
7.  `sum.print();`: 命令将处理后的数据打印出来。
8.  `env.execute("Tumbling Window Example");`: 启动Flink任务。

基于计数的滚动窗口：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> text = env.socketTextStream("localhost", 9999); DataStream<Tuple2<String, Integer>> counts = text .flatMap(new Tokenizer()) .keyBy(0) .countWindow(5) // Count window of 5 elements .sum(1); counts.print().setParallelism(1); env.execute("Window WordCount"); } public static final class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> { @Override public void flatMap(String value, Collector<Tuple2<String, Integer>> out) { String[] words = value.toLowerCase().split("\\W+"); for (String word : words) { if (word.length() > 0) { out.collect(new Tuple2<>(word, 1)); } } } }
```

这段程序从本地9999端口读取数据流，对每一行的单词进行小写处理和分割，然后在滑动窗口中（大小为5个元素）计算出各个单词的出现次数。

### 滑动窗口

滑动窗口的大小固定，但窗口之间不是首尾相接，会有部分重合。同样，滑动窗口也可以基于时间和计数定义。

滑动窗口的参数有两个：**窗口大小和滑动步长**

![[Blog/Picture/de342b5651b9d22ae83205bdcfff8535_MD5.png]]

基于时间的滑动窗口：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> input = env.socketTextStream("localhost", 9999); DataStream<Tuple2<String, Integer>> processedInput = input.map(new MapFunction<String, Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> map(String value){ String[] words = value.split(","); return new Tuple2<>(words[0], Integer.parseInt(words[1])); } }); // 指定窗口类型为滑动窗口，窗口大小为10分钟，滑动步长为5分钟 DataStream<Tuple2<String, Integer>> windowCounts = processedInput .keyBy(0) .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5))) .reduce(new ReduceFunction<Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2){ return new Tuple2<>(value1.f0, value1.f1 + value2.f1); } }); windowCounts.print().setParallelism(1); env.execute("Time Window Example"); }
```

这段程序从一个套接字端口读取输入数据，将每行输入按照“,”切分并映射为tuple（字符串，整数）。然后，它按照第一个元素（即字符串）进行分组，并使用滑动窗口（窗口大小为10秒，滑动步长为5秒）进行聚合 - 在每个窗口内，所有具有相同键的值的整数部分被相加。最终结果会在控制台上打印。

基于计数的滑动窗口：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> text = env.socketTextStream("localhost", 9999); DataStream<Tuple2<String, Integer>> counts = text .flatMap(new Tokenizer()) .keyBy(0) .countWindow(5, 1) .sum(1); counts.print().setParallelism(1); env.execute("Sliding Window WordCount"); } public static final class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> { @Override public void flatMap(String value, Collector<Tuple2<String, Integer>> out) { String[] words = value.toLowerCase().split("\\W+"); for (String word : words) { if (word.length() > 0) { out.collect(new Tuple2<>(word, 1)); } } } }
```

这段代码是实时滑动窗口词频统计程序。它从本地9999端口读取数据流，将接收到的每行文本拆分为单词然后输出为(单词,1)的形式，接着按照单词分组，使用大小为5，步长为1的滑动窗口，并对每个窗口中的同一单词出现次数进行求和，最后打印结果。

### 会话窗口

**会话窗口是Flink中一种基于时间的窗口类型，每个窗口的大小不固定，且相邻两个窗口之间没有重叠，“会话”终止的标志就是隔一段时间没有数据进来**

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<Tuple2<String, Long>> inputStream = env.fromElements( new Tuple2<>("user1", 1617229200000L), new Tuple2<>("user1", 1617229205000L), new Tuple2<>("user2", 1617229210000L), new Tuple2<>("user1", 1617229215000L), new Tuple2<>("user2", 1617229220000L) ); SingleOutputStreamOperator<Tuple2<String, Long>> resultStream = inputStream .keyBy(value -> value.f0) .window(EventTimeSessionWindows.withGap(Time.minutes(5))) .sum(1); resultStream.print(); env.execute("Session Window Example"); }
```

这段代码从一个数据流中读取用户活动数据（包含用户ID和Unix时间戳），然后根据用户ID将数据进行分组，并应用了一个会话窗口（当用户五分钟内无活动则关闭该用户的窗口）。

然后，它对每个用户在各自窗口内的活动时间戳求和，并打印出结果。最后执行的名为"Session Window Example"的任务即完成了这一流式计算过程。

### 按键分区窗口和非按键分区窗口

在Flink中，数据流可以按键分区（keyed）和非按键分区（non-keyed）。

按键分区是指将数据流根据特定的键值进行分区，使得相同键值的元素被分配到同一个分区中。这样可以保证相同键值的元素由同一个worker实例处理。只有按键分区的数据流才能使用键分区状态和计时器。

非按键分区是指数据流没有根据特定的键值进行分区。这种情况下，数据流中的元素可以被任意分配到不同的分区中。

在定义窗口操作之前，首先需要确定，到底是基于按键分区（Keyed）来开窗，还是直接在没有按键分区的DataStream上开窗。也就是在调用窗口算子之前是否有keyBy操作。

按键分区窗口：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> text = env.socketTextStream("localhost", 9999); DataStream<Tuple2<String, Integer>> counts = // 将输入字符串拆分为tuple类型，包含word和数量 text.map(new MapFunction<String, Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> map(String value) { return new Tuple2<>(value, 1); } }) // 根据元组的第一字段（word）进行分区键 .keyBy(0) // 定义一个滚动窗口，时间间隔为5秒 .window(TumblingProcessingTimeWindows.of(Time.seconds(5))) // 应用reduce函数，累加各个窗口中同一单词的数量 .reduce(new ReduceFunction<Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) { return new Tuple2<>(value1.f0, value1.f1 + value2.f1); } }); counts.print(); env.execute("Window WordCount");
```

这段代码从 localhost 的 9999 端口接收数据流，将输入的每个字符串作为一个单词和数字 1 的 tuple 对象，然后根据单词进行分区，创建一个滚动窗口（间隔为5秒），并在每个窗口中对同一单词的数量进行累加统计，最后打印出结果。

非按键分区窗口：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> text = env.socketTextStream("localhost", 9999); DataStream<Integer> parsed = text.map(new MapFunction<String, Integer>() { @Override public Integer map(String value) { return Integer.parseInt(value); } }); DataStream<Integer> windowCounts = parsed .windowAll(TumblingProcessingTimeWindows.of(Time.seconds(5))) .reduce(new ReduceFunction<Integer>() { @Override public Integer reduce(Integer value1, Integer value2) { return value1 + value2; } }); windowCounts.print().setParallelism(1); env.execute("Non keyed Window example"); }
```

这段程序从localhost的9999端口读取数据流，把每条数据转化为整数，然后在5秒的滚动窗口内将所有的整数值进行累加，并打印出结果。

### 窗口函数（WindowFunction）

**所谓的“窗口函数”（window functions），就是定义窗口如何进行计算操作的函数**

窗口函数根据处理的方式可以分为两类：「**增量窗口聚合函数**」和「**全窗口聚合函数**」。

#### 增量窗口聚合函数

增量窗口聚合函数每来一条数据就立即进行计算，中间保持着聚合状态，但是不立即输出结果，等到窗口到了结束时间需要输出计算结果的时候，取出之前聚合的状态直接输出。

常见的增量聚合函数有：`reduce()`、`aggregate()`、`sum()`、`min()`、`max()`。

下面是一个使用增量聚合函数的代码示例：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<Long> data = env.fromSequence(1,Long.MAX_VALUE); DataStream<Long> result = data.keyBy(value -> value % 2) .window(TumblingProcessingTimeWindows.of(Time.seconds(5))) .aggregate(new SumAggregator()); result.print(); env.execute("Incremental Aggregation Job"); } public static class SumAggregator implements AggregateFunction<Long, Long, Long> { @Override public Long createAccumulator() { return 0L; } @Override public Long add(Long value, Long accumulator) { return value + accumulator; } @Override public Long getResult(Long accumulator) { return accumulator; } @Override public Long merge(Long a, Long b) { return a + b; } }
```

这段代码从1到`Long.MAX_VALUE`产生一个连续的数据流。接着，它将数据按照奇偶性进行分类，并在每个5秒的时间窗口内对相同类别的数值进行累加操作。最后打印出累加结果。

#### 全窗口函数

全窗口函数是指在整个窗口中的所有数据都准备好后才进行计算。

Flink中的全窗口函数有两种： `WindowFunction`和`ProcessWindowFunction` 。

与增量窗口函数不同，全窗口函数可以访问窗口中的所有数据，因此可以执行更复杂的计算。例如，可以计算窗口中数据的中位数，或者对窗口中的数据进行排序。

WindowFunction接收一个Iterable类型的输入，其中包含了窗口中所有的数据。ProcessWindowFunction则更加强大，它不仅可以访问窗口中的所有数据， 还可以获取到一个“上下文对象”（Context）。

这个上下文对象非常强大，不仅能够获取窗口信息，还可以访问当前的时间和状态信息。这里的时间就包括了处理时间（Processing Time）和事件时间水位线（Event Time Watermark）。

这就使得 ProcessWindowFunction 更加灵活、功能更加丰富，WindowFunction作用可以被 ProcessWindowFunction 全覆盖。

不过这种额外的功能可能会带来一些性能上的损失，因此只有当你确实需要这些额外功能时，才应该使用ProcessWindowFunction，如果你不需要这些功能，“简单”的WindowFunction可能会更有效率。

下面是使用 WindowFunction 计算窗口内数据总和的代码示例：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> text = env.fromElements("a", "b", "c", "a", "b", "b"); DataStream<String> withTimestampsAndWatermarks = text.assignTimestampsAndWatermarks( WatermarkStrategy.<String>forBoundedOutOfOrderness(Duration.ofMillis(100)) .withTimestampAssigner((event, timestamp) -> System.currentTimeMillis()) ); DataStream<Tuple2<String, Integer>> mapped = withTimestampsAndWatermarks.map( new MapFunction<String, Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> map(String value) { return new Tuple2<>(value, 1); } }); mapped.keyBy(0) .timeWindow(Time.seconds(5)) .apply(new SumWindowFunction()) .print(); env.execute("Window Sum"); }
```

下面是一个使用ProcessWindowFunction统计网站1天UV的代码示例：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<Tuple2<String, Integer>> data = env.fromElements( new Tuple2<>("user1", 1), new Tuple2<>("user2", 1), new Tuple2<>("user1", 1)); data = data.assignTimestampsAndWatermarks(WatermarkStrategy .<Tuple2<String,Integer>>forMonotonousTimestamps() .withTimestampAssigner((event, timestamp) -> System.currentTimeMillis()) ); data.keyBy(0) .window(TumblingEventTimeWindows.of(Time.days(1))) .process(new UVProcessWindowFunction()) .print(); env.execute("Daily User View Count"); } public static class UVProcessWindowFunction extends ProcessWindowFunction<Tuple2<String, Integer>, Tuple2<String, Long>, Tuple, TimeWindow> { @Override public void process(Tuple key, Context context, Iterable<Tuple2<String, Integer>> input, Collector<Tuple2<String, Long>> out){ long count = 0; BloomFilter<CharSequence> bloomFilter = BloomFilter.create(Funnels.stringFunnel(), 100000, 0.01); for (Tuple2<String, Integer> in: input) { if (!bloomFilter.mightContain(in.f0)) { count += 1; bloomFilter.put(in.f0); } } out.collect(new Tuple2<>(key.getField(0), count)); } }
```

这段代码从数据流中读取用户视图数据（数据为("user", view\_count)），然后对每个用户的观看次数实现了基于时间窗口（一天）的统计。利用布隆过滤器并在窗口内去重，可以避免重复计数。最后，每个窗口结束时，它会输出每个用户的id和相应的不重复观看次数。

#### 增量窗口函数和全窗口函数结合使用

全窗口函数为处理提供了更多的背景信息，因为它需要等到收集完所有窗口内的数据才进行计算，但是全窗口函数可能会增加系统的复杂性和运行时间。

另一方面，增量窗口函数可以在数据进入窗口时进行部分聚合计算，从而提高效率，但是它可能不适用于所有类型的计算，例如中位数或者标准差这种需要全部数据的计算就无法使用增量聚合。

在实际应用中，我们往往希望兼具这两者的优点，把它们结合在一起使用。Flink 的**Window API** 就给我们实现了这样的用法。

之前在调用 WindowedStream 的`reduce()`和`aggregate()`方法时，只是简单地直接传入了一个 ReduceFunction 或 AggregateFunction 进行增量聚合。除此之外，其实还可以传入第二个参数：一个全窗口函数，可以是 `WindowFunction` 或者`ProcessWindowFunction`。

```java
// ReduceFunction 与 WindowFunction 结合 public <R> SingleOutputStreamOperator<R> reduce(ReduceFunction<T> reduceFunction, WindowFunction<T, R, K, W> function) // ReduceFunction 与 ProcessWindowFunction 结合 public <R> SingleOutputStreamOperator<R> reduce(ReduceFunction<T> reduceFunction, ProcessWindowFunction<T, R, K, W> function) // AggregateFunction 与 WindowFunction 结合 public <ACC, V, R> SingleOutputStreamOperator<R> aggregate(AggregateFunction<T, ACC, V> aggFunction, WindowFunction<V, R, K, W> windowFunction) // AggregateFunction 与 ProcessWindowFunction 结合 public <ACC, V, R> SingleOutputStreamOperator<R> aggregate(AggregateFunction<T, ACC, V> aggFunction, ProcessWindowFunction<V, R, K, W> windowFunction)
```

这样调用的处理机制是：基于第一个参数（增量聚合函数）来处理窗口数据，每来一个数据就做一次聚合；等到窗口需要触发计算时，则调用第二个参数（全窗口函数）的处理逻辑输出结果。

**需要注意的是，这里的全窗口函数就不再缓存所有数据了，而是直接将增量聚合函数的结果拿来当作了 Iterable 类型的输入。一般情况下，这时的可迭代集合中就只有一个元素了**

下面我们举一个具体的实例来说明：

在网站的各种统计指标中，一个很重要的统计指标就是热门的链接，想要得到热门的 url，前提是得到每个链接的“热门度”。一般情况下，可以用url 的浏览量（点击量）表示热门度。我们这里统计 10 秒钟的 url 浏览量，每 5 秒钟更新一次。

我们可以定义滑动窗口，并结合增量聚合函数和全窗口函数来得到统计结果，代码示例如下：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> text = env.socketTextStream("localhost", 9999); DataStream<Tuple2<String, Long>> urlCounts = text .flatMap(new Tokenizer()) .keyBy(value -> value.f0) .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5))) .aggregate(new CountAgg(), new WindowResultFunction()); urlCounts.print(); env.execute("UrlCount Job"); } public static class CountAgg implements AggregateFunction<Tuple2<String, Integer>, Long, Long> { @Override public Long createAccumulator() { return 0L; } @Override public Long add(Tuple2<String, Integer> value, Long accumulator) { return accumulator + value.f1; } @Override public Long getResult(Long accumulator) { return accumulator; } @Override public Long merge(Long a, Long b) { return a + b; } } public static class WindowResultFunction implements WindowFunction<Long, Tuple2<String, Long>, String, TimeWindow> { @Override public void apply(String key, TimeWindow window, Iterable<Long> input, Collector<Tuple2<String, Long>> out) { Long count = input.iterator().next(); out.collect(new Tuple2<>(key, count)); } } public static final class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> { @Override public void flatMap(String value, Collector<Tuple2<String, Integer>> out) { String[] words = value.toLowerCase().split("\\W+"); for (String word : words) { if (word.length() > 0) { out.collect(new Tuple2<>(word, 1)); } } } }
```

在这个示例中，我们首先把数据根据 URL 进行了分组 (keyBy)，然后定义了一个滑动窗口，窗口长度是10秒，每5秒滑动一次。接着我们使用增量聚合函数 `CountAgg` 对每个窗口内的元素进行聚合，最后用全窗口函数 `WindowResultFunction` 输出结果。

### Window重叠优化

窗口重叠是指在使用滑动窗口时，多个窗口之间存在重叠部分。这意味着同一批数据可能会被多个窗口同时处理。

例如，假设我们有一个数据流，它包含了0到9的整数。我们定义了一个大小为5的滑动窗口，滑动距离为2。那么，我们将会得到以下三个窗口：

-   窗口1：包含0, 1, 2, 3, 4
-   窗口2：包含2, 3, 4, 5, 6
-   窗口3：包含4, 5, 6, 7, 8

在这个例子中，窗口1和窗口2之间存在重叠部分，即2, 3, 4。同样，窗口2和窗口3之间也存在重叠部分，即4, 5, 6。

`enableOptimizeWindowOverlap()`方法是用来启用Flink的窗口重叠优化功能的。它可以减少计算重叠窗口时的计算量。

在我之前给出的代码示例中，我没有使用`enableOptimizeWindowOverlap()`方法来启用窗口重叠优化功能。这意味着Flink不会尝试优化计算重叠窗口时的计算量。

如果你想使用窗口重叠优化功能，你可以在你的代码中添加以下行：

```undefined
env.getConfig().enableOptimizeWindowOverlap();
```

这将启用窗口重叠优化功能，Flink将尝试优化计算重叠窗口时的计算量。

## 触发器（Trigger）

触发器主要是用来控制窗口什么时候触发计算。所谓的“触发计算”，本质上就是执行窗口函数，所以可以认为是计算得到结果并输出的过程。

基于 WindowedStream 调用`trigger()`方法，就可以传入一个自定义的窗口触发器（Trigger）。

```css
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); DataStream<String> dataStream = env.socketTextStream("localhost", 9999); dataStream.map(new MapFunction<String, Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> map(String value) { return new Tuple2<>(value, 1); } }) .keyBy(value -> value.f0) .window(TumblingProcessingTimeWindows.of(Time.seconds(5))) .trigger(new MyTrigger()) .sum(1) .print(); env.execute("Flink Trigger Example"); } public static class MyTrigger extends Trigger<Tuple2<String, Integer>, TimeWindow> { @Override public TriggerResult onElement(Tuple2<String, Integer> stringIntegerTuple2, long l, TimeWindow timeWindow, TriggerContext triggerContext) throws Exception { return TriggerResult.FIRE_AND_PURGE; } @Override public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx) { return TriggerResult.CONTINUE; } @Override public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) { return TriggerResult.CONTINUE; } @Override public void clear(TimeWindow window, TriggerContext ctx) { } }
```

这段代码主要从localhost的9999端口读取数据流，每条数据映射为一个包含该数据和整数1的元组。然后按照元组的第一个元素进行分组，并在每5秒的滚动窗口中对元组的第二个元素求和。最后使用用户自定义触发器，当新元素到达时立即触发计算并清空窗口，但在处理时间或事件时间上不做任何操作。

Trigger 是窗口算子的内部属性，每个窗口分配器（WindowAssigner）都会对应一个默认的触发器。

对于 Flink 内置的窗口类型，它们的触发器都已经做了实现。例如，所有事件时间窗口，默认的触发器都是EventTimeTrigger，类似还有 ProcessingTimeTrigger 和 CountTrigger。所以一般情况下是不需要自定义触发器的，这块了解一下即可。

## 移除器（Evictor）

移除器（Evictor）是用于在滚动窗口或会话窗口中控制数据保留和清理的组件。它可以根据特定的策略从窗口中删除一些数据，以确保窗口中保留的数据量不超过指定的限制。

移除器通常与窗口分配器一起使用，窗口分配器负责确定数据属于哪个窗口，而移除器则负责清理窗口中的数据。

以下是一个使用移除器的代码示例，演示如何在滚动窗口中使用基于计数的移除器：

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); // 创建一个包含整数和时间戳的流 DataStream<Tuple2<Integer, Long>> dataStream = env.fromElements( Tuple2.of(1, System.currentTimeMillis()), Tuple2.of(2, System.currentTimeMillis() + 1000), Tuple2.of(3, System.currentTimeMillis() + 2000), Tuple2.of(4, System.currentTimeMillis() + 3000), Tuple2.of(5, System.currentTimeMillis() + 4000), Tuple2.of(6, System.currentTimeMillis() + 5000) ); // 添加以下代码设置水印和事件时间戳 dataStream = dataStream.assignTimestampsAndWatermarks(WatermarkStrategy.<Tuple2<Integer, Long>>forBoundedOutOfOrderness(Duration.ofSeconds(1)) .withTimestampAssigner((event, timestamp) -> event.f1)); // 在滚动窗口中使用基于计数的移除器，保留最近3个元素 dataStream .keyBy(value -> value.f0) .window(TumblingEventTimeWindows.of(Time.seconds(5))) .trigger(CountTrigger.of(3)) .evictor(CountEvictor.of(3)) .aggregate(new MyAggregateFunction(), new MyProcessWindowFunction()) .print(); env.execute("Flink Evictor Example"); } // 自定义聚合函数 private static class MyAggregateFunction implements AggregateFunction<Tuple2<Integer, Long>, Integer, Integer> { @Override public Integer createAccumulator() { return 0; } @Override public Integer add(Tuple2<Integer, Long> value, Integer accumulator) { return accumulator + 1; } @Override public Integer getResult(Integer accumulator) { return accumulator; } @Override public Integer merge(Integer a, Integer b) { return a + b; } } // 自定义处理窗口函数 private static class MyProcessWindowFunction extends ProcessWindowFunction<Integer, String, Integer, TimeWindow> { private transient ListState<Integer> countState; @Override public void open(Configuration parameters) throws Exception { super.open(parameters); ListStateDescriptor<Integer> descriptor = new ListStateDescriptor<>("countState", Integer.class); countState = getRuntimeContext().getListState(descriptor); } @Override public void process(Integer key, Context context, Iterable<Integer> elements, Collector<String> out) throws Exception { int count = elements.iterator().next(); countState.add(count); long windowStart = context.window().getStart(); long windowEnd = context.window().getEnd(); String result = "Window: " + windowStart + " to " + windowEnd + ", Count: " + countState.get().iterator().next(); out.collect(result); } }
```

这段代码主要用于对一串包含整数和时间戳的元素进行处理。首先，它创建了一个流并赋予了水印和时间戳。然后在滚动窗口中使用基于计数的触发器和驱逐器，只保留最近的三个元素。之后，通过自定义聚合和窗口函数，来处理窗口内的数据，聚合函数计算每个窗口内元素的数量，窗口函数将结果与窗口的开始和结束时间一起输出。

## Flink Time 时间语义

Flink定义了三类时间

-   **事件时间（Event Time）**：数据在数据源产生的时间，一般由事件中的时间戳描述，比如用户日志中的TimeStamp。
-   **摄取时间（Ingestion Time）**：数据进入Flink的时间，记录被Source节点观察到的系统时间。
-   **处理时间（Process Time）**：数据进入Flink被处理的系统时间（Operator处理数据的系统时间）。

![[Blog/Picture/b298f2b4630f0cb71f7062f31eb28e60_MD5.png]]

Flink 流式计算的时候需要显示定义时间语义，根据不同的时间语义来处理数据，比如指定的时间语义是事件时间，那么我们就要切换到事件时间的世界观中，窗口的起始与终止时间都是以事件时间为依据。

在Flink中默认使用的是Process Time，如果要使用其他的时间语义，在执行环境中可以进行设置。

```java
//设置时间语义为Ingestion Time env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime); //设置时间语义为Event Time 我们还需要指定一下数据中哪个字段是事件时间（下文会讲） env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

### Watermark(水印)

**Watermark的本质实质上是时间戳，简单而言，它是用来处理迟到数据的**

在使用Flink处理数据的时候，数据通常都是按照事件产生的时间（事件时间）的顺序进入到Flink，但是在遇到特殊情况下，比如遇到网络延迟或者使用Kafka（多分区） 很难保证数据都是按照事件时间的顺序进入Flink，很有可能是乱序进入。

如果数据一旦是乱序进入，那么在使用Window处理数据的时候，就会出现延迟数据不会被计算的问题。

-   举例： 滚动窗口长度10s。
    
    2020-04-25 10:00:01
    
    2020-04-25 10:00:02
    
    2020-04-25 10:00:03
    
    2020-04-25 10:00:11 窗口触发执行
    
    2020-04-25 10:00:05 延迟数据，不会被上个窗口所计算，导致计算结果不正确
    

如果有延迟数据，那么窗口需要等待全部的数据到来之后，再触发窗口执行。

需要等待多久？不可能无限期等待，我们用户可以自己来设置延迟时间，这样就可以尽可能保证延迟数据被处理。

使用Watermark就可以很好的解决延迟数据的问题。

根据用户指定的延迟时间生成水印（Watermak = 最大事件时间-指定延迟时间），当 Watermak 大于等于窗口的停止时间，这个窗口就会被触发执行。

-   举例：滚动窗口长度10s，指定延迟时间3s
    
    2020-04-25 10:00:01 wm:2020-04-25 09:59:58
    
    2020-04-25 10:00:02 wm:2020-04-25 09:59:59
    
    2020-04-25 10:00:03 wm:2020-04-25 10:00:00
    
    2020-04-25 10:00:09 wm:2020-04-25 10:00:06
    
    2020-04-25 10:00:12 wm:2020-04-25 10:00:09
    
    2020-04-25 10:00:08 wm:2020-04-25 10:00:05 延迟数据
    
    2020-04-25 10:00:13 wm:2020-04-25 10:00:10
    

**如果没有 Watermark ，那么在倒数第三条数据来的时候，就会触发执行，倒数第二条的延迟数据就不会被计算，有了水印之后就可以处理延迟3s内的数据**

#### 生成水印策略

-   **周期性水印（Periodic Watermark）**：根据事件或者处理时间周期性的触发水印生成器(Assigner)，默认100ms，每隔100毫秒自动向流里注入一个Watermark。
    
    ```java
    final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime); env.getConfig().setAutoWatermarkInterval(100); DataStream<String> stream = env.socketTextStream("node01", 8888) .assignTimestampsAndWatermarks(WatermarkStrategy .<String>forBoundedOutOfOrderness(Duration.ofSeconds(3)) .withTimestampAssigner((event, timestamp) -> { return Long.parseLong(event.split(" ")[0]); }));
    ```
    
-   **间歇性水印**：间歇性水印（Punctuated Watermark）在观察到事件后，会依据用户指定的条件来决定是否发射水印。
    
    ```java
    public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<Tuple2<String, Long>> { @Override public long extractTimestamp(Tuple2<String, Long> element, long previousElementTimestamp) { return element.f1; } @Override public Watermark checkAndGetNextWatermark(Tuple2<String, Long> lastElement, long extractedTimestamp) { return lastElement.f0.equals("watermark") ? new Watermark(extractedTimestamp) : null; } public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.addSource(new SourceFunction<Tuple2<String, Long>>() { private boolean running = true; @Overrid public void run(SourceContext<Tuple2<String, Long>> ctx) throws Exception { while (running) { long currentTimestamp = System.currentTimeMillis(); ctx.collect(new Tuple2<>("key", currentTimestamp)); if (currentTimestamp % 10 == 0) { // 每隔一段时间发出一个含有"watermark"的特殊事件 ctx.collect(new Tuple2<>("watermark", currentTimestamp)); } Thread.sleep(1000); } } @Override public void cancel() { running = false; } }).assignTimestampsAndWatermarks(new PunctuatedAssigner()) .print(); env.execute("Punctuated Watermark Example"); } }
    ```
    

这段代码定义了一个名为PunctuatedAssigner的时间戳和watermark分配器类，用于从接收到的元素中提取出时间戳，并根据特定条件（在本例中，元素的key是否为"watermark"）生成并发送watermark。

在main方法中，创建了一个源函数，此函数每秒生成一个新的事件，并且每隔10毫秒就发出一个包含"watermark"的特殊事件。这些事件被收集，分配时间戳和watermark，然后打印出来。

### 允许延迟（Allowed Lateness）

Flink 还提供了另外一种方式处理迟到数据。我们可以将未收入窗口的迟到数据，放入“侧输出流”（side output）进行另外的处理。所谓的侧输出流，相当于是数据流的一个“分支”，**这个流中单独放置那些错过了、本该被丢弃的数据**。

此方法需要传入一个“输出标签”（OutputTag），用来标记分支的迟到数据流。因为保存的就是流中的原始数据，所以 OutputTag 的类型与流中数据类型相同：

```dart
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); // 定义 OutputTag 来标识侧输出流 final OutputTag<String> lateDataTag = new OutputTag<String>("late-data"){}; DataStream<String> dataStream = env.socketTextStream("localhost", 9000); SingleOutputStreamOperator<Tuple2<String, Integer>> resultStream = dataStream .map(new MapFunction<String, Tuple2<String, Integer>>() { @Override public Tuple2<String, Integer> map(String value) throws Exception { return new Tuple2<>(value, 1); } }) .keyBy(value -> value.f0) .process(new ProcessFunction<Tuple2<String, Integer>, Tuple2<String, Integer>>() { @Override public void processElement(Tuple2<String, Integer> value, Context ctx, Collector<Tuple2<String, Integer>> out) throws Exception { if (value.f1 == 1) { out.collect(value); } else { // 将迟到的数据发送到侧输出流 ctx.output(lateDataTag, "Late data detected: " + value); } } }); // 获取侧输出流 DataStream<String> lateDataStream = resultStream.getSideOutput(lateDataTag); resultStream.print(); lateDataStream.print(); env.execute("SideOutput Example"); }
```

这段代码首先建立一个从本地 9000 端口读取数据的流，然后将每一行数据映射为一个二元组 (value, 1)。接着按照第一个字段进行分组，并进行处理：如果二元组的第二个元素等于 1，则直接输出；否则，该条数据会被视为“迟到数据”并输出至侧输出流。最后，主流和侧输出流的结果都会打印出来。

## Flink关联维度表

在Flink实际开发过程中，可能会遇到 source 进来的数据，需要连接数据库里面的字段，再做后面的处理，比如，想要通过id获取对应的地区名字，这时候需要通过id查询地区维度表，获取具体的地区名。

对于不同的应用场景，关联维度表的方式不同

-   场景1：维度表信息基本不发生改变，或者发生改变的频率很低。
    
    实现方案：采用Flink提供的CachedFile。
    
    Flink提供了一个分布式缓存（CachedFile），可以使用户在并行函数中很方便的读取本地文件，并把它放在TaskManager节点中，防止Task重复拉取。
    
    此缓存的工作机制如下：程序注册一个文件或者目录(本地或者远程文件系统，例如hdfs或者s3)，通过ExecutionEnvironment注册缓存文件并为它起一个名称。
    
    当程序执行，Flink自动将文件或者目录复制到所有TaskManager节点的本地文件系统，仅会执行一次。用户可以通过这个指定的名称查找文件或者目录，然后从TaskManager节点的本地文件系统访问它。
    
    ```java
    public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.registerCachedFile("/root/id2city", "id2city"); DataStream<String> socketStream = env.socketTextStream("node01", 8888); DataStream<Integer> stream = socketStream.map(Integer::valueOf); DataStream<String> result = stream.map(new RichMapFunction<Integer, String>() { private Map<Integer, String> id2CityMap; @Override public void open(Configuration parameters) throws Exception { super.open(parameters); id2CityMap = new HashMap<>(); BufferedReader reader = new BufferedReader(new FileReader(getRuntimeContext().getDistributedCache().getFile("id2city"))); String line; while ((line = reader.readLine()) != null) { String[] splits = line.split(" "); Integer id = Integer.parseInt(splits[0]); String city = splits[1]; id2CityMap.put(id, city); } reader.close(); } @Override public String map(Integer value) throws IOException { return id2CityMap.getOrDefault(value, "not found city"); } }); result.print(); env.execute(); }
    ```
    
    这段程序首先从"node01"主机的8888端口读取数据，然后将其转换为整数流。接着，它用一个富映射函数（RichMapFunction）将每个整数ID映射到城市名。这个映射是从在"/root/id2city"路径下注册的缓存文件中读取的。如果无法找到某个ID对应的城市，就会返回"not found city"。
    
    在集群中查看对应TaskManager的log日志，发现注册的file会被拉取到各个TaskManager的工作目录区。
    
-   场景2：对于维度表更新频率比较高并且对于查询维度表的实时性要求比较高。
    
    实现方案：使用定时器，定时加载外部配置文件或者数据库
    
    ```java
    public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.setParallelism(1); DataStream<String> stream = env.socketTextStream("node01", 8888); stream.map(new RichMapFunction<String, String>() { private HashMap<String,String> map = new HashMap<>(); @Override public void open(Configuration parameters) throws Exception { System.out.println("init data ..."); query(); Timer timer = new Timer(true); timer.schedule(new TimerTask() { @Override public void run() { try { query(); } catch (IOException e) { e.printStackTrace(); } } },1000,2000); } void query() throws IOException { Path path = Paths.get("D:\\code\\StudyFlink\\data\\id2city"); Stream<String> lines = Files.lines(path); lines.forEach(line -> { String[] parts = line.split(" "); map.put(parts[0], parts[1]); }); lines.close(); } @Override public String map(String key) throws Exception { return map.getOrDefault(key, "not found city"); } }).print(); env.execute(); }
    ```
    
    这段代码从名为"node01"的服务器的8888端口读取数据流，然后通过映射函数将每个接收到的数据键值（假设是城市ID）转换为对应的城市名称。此映射来自一个定期更新的文件"D:\\code\\StudyFlink\\data\\id2city"，如果没有找到匹配的城市ID，则返回"not found city"。
    
-   场景3：对于维度表更新频率高并且对于查询维度表的实时性要求较高。
    
    实现方案：将更改的信息同步至Kafka配置Topic中，然后将kafka的配置流信息变成广播流，广播到业务流的各个线程中。
    

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); Properties props = new Properties(); props.setProperty("bootstrap.servers", "node01:9092,node02:9092,node03:9092"); props.setProperty("group.id", "flink-kafka-001"); props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer"); props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer"); FlinkKafkaConsumer<String> consumer = new FlinkKafkaConsumer<>( "configure", new SimpleStringSchema(), props ); consumer.setStartFromLatest(); DataStream<String> configureStream = env.addSource(consumer); DataStream<String> busStream = env.socketTextStream("node01", 8888); MapStateDescriptor<String, String> descriptor = new MapStateDescriptor<>( "dynamicConfig", BasicTypeInfo.STRING_TYPE_INFO, BasicTypeInfo.STRING_TYPE_INFO ); BroadcastStream<String> broadcastStream = configureStream.broadcast(descriptor); busStream.connect(broadcastStream).process( new BroadcastProcessFunction<String, String, String>() { @Override public void processElement(String line, ReadOnlyContext ctx, Collector<String> out) throws Exception { String city = ctx.getBroadcastState(descriptor).get(line); if (city == null) { out.collect("not found city"); } else { out.collect(city); } } @Override public void processBroadcastElement(String line, Context ctx, Collector<String> out) throws Exception { String[] elems = line.split(" "); ctx.getBroadcastState(descriptor).put(elems[0], elems[1]); } } ).print(); env.execute(); }
```

这段代码将从Kafka中获取的数据作为广播流，然后与从socket中获取的数据处理。在处理过程中，根据socket中的数据（作为key）查找广播状态中的城市名称（作为value），如果找到，则输出城市名，否则输出"not found city"。其中，Kafka中的数据以空格分隔，第一个元素作为key，第二个元素作为value存入BroadcastState。

## Table API & Flink SQL

在Spark中有DataFrame这样的关系型编程接口，因其强大且灵活的表达能力，能够让用户通过非常丰富的接口对数据进行处理，有效降低了用户的使用成本。

Flink也提供了关系型编程接口Table API以及基于Table API的SQL API，让用户能够通过使用结构化编程接口高效地构建Flink应用。同时Table API以及SQL能够统一处理批量和实时计算业务，无须切换修改任何应用代码就能够基于同一套API编写流式应用和批量应用，从而达到真正意义的流批统一。

![[Blog/Picture/38df98fd902e0d5c40d71962729ec3ce_MD5.png]]

在 Flink 1.8 架构里，如果用户需要同时流计算、批处理的场景下，用户需要维护两套业务代码，开发人员也要维护两套技术栈，非常不方便。 Flink 社区很早就设想过将批数据看作一个有界流数据，将批处理看作流计算的一个特例，从而实现流批统一。

阿里巴巴的 Blink 团队在这方面做了大量的工作，已经实现了 Table API & SQL 层的流批统一。阿里巴巴已经将 Blink 开源回馈给 Flink 社区。

### 开发环境构建

在 Flink 1.9 中，Table 模块迎来了核心架构的升级，引入了阿里巴巴Blink团队贡献的诸多功能，取名叫： **Blink Planner**。

在使用Table API和SQL开发Flink应用之前，通过添加Maven的依赖配置到项目中，在本地工程中引入相应的依赖库，库中包含了Table API和SQL接口。

```xml
<dependency> <groupId>org.apache.flink</groupId> <artifactId>flink-table-planner_2.12</artifactId> <version>1.13.6</version> </dependency> <dependency> <groupId>org.apache.flink</groupId> <artifactId>flink-table-api-scala-bridge_2.12</artifactId> <version>1.13.6</version> </dependency>
```

### Table Environment

和DataStream API一样，Table API和SQL具有相同的基本编程模型。首先需要构建对应的 TableEnviroment 创建关系型编程环境，才能够在程序中使用Table API和SQL来编写应用程序，另外Table API和SQL接口可以在应用中同时使用，Flink SQL基于Apache Calcite框架实现了SQL标准协议，是构建在Table API之上的更高级接口。

首先需要在环境中创建 TableEnvironment 对象，TableEnvironment 中提供了注册内部表、执行Flink SQL语句、注册自定义函数等功能。根据应用类型的不同，TableEnvironment 创建方式也有所不同，但是都是通过调用`create()`方法创建。

流计算环境下创建 TableEnviroment ：

```java
//创建流式计算的上下文环境 final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); //创建Table API的上下文环境 StreamTableEnvironment streamTableEnvironment = StreamTableEnvironment.create(env);
```

### Table API

**Table API 顾名思义，就是基于“表”（Table）的一套 API，专门为处理表而设计的**

它提供了关系型编程模型，可以用来处理结构化数据，支持表和视图的概念。在此基础上，Flink 还基于 Apache Calcite 实现了对 SQL 的支持。这样一来，我们就可以在 Flink 程序中直接写 SQL 来实现需求了，非常实用。

下面是一个简单的例子，它使用Java编写了一个Flink程序，该程序使用 Table API 从CSV文件中读取数据，然后执行简单的查询并将结果写入到自定义的Sink中。

首先我们需要导入maven依赖：

```xml
<dependency> <groupId>org.apache.flink</groupId> <artifactId>flink-table-api-java-bridge_2.12</artifactId> <version>1.13.6</version> </dependency>
```

代码示例如下：

```undefined
public static void main(String[] args) throws Exception { // 创建流处理环境 final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); // 创建表环境 EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build(); StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, settings); // 从CSV文件中读取数据 DataStream<Tuple2<String, Integer>> data = env.readTextFile("input.csv") .map(line -> { String[] parts = line.split(","); return new Tuple2<>(parts[0], Integer.parseInt(parts[1])); }) .returns(Types.TUPLE(Types.STRING, Types.INT)); // 使用Table API将数据转换为表并注册为视图 String name = "people"; Schema schema = Schema.newBuilder() .column("name", DataTypes.STRING()) .column("age", DataTypes.INT()) .build(); tableEnv.createTemporaryView(name, data, schema); // 使用SQL查询年龄大于30的人 Table result = tableEnv.sqlQuery("SELECT name, age FROM people WHERE age > 30"); // 将结果转换为DataStream DataStream<Row> output = tableEnv.toDataStream(result); output.addSink(new SinkFunction<Row>() { @Override public void invoke(Row value, Context context) throws Exception { // implement the sink here, e.g., write into a file, send to Kafka, etc. } }); env.execute(); }
```

这段代码是在流处理环境中实现的一个简单的ETL（提取-转换-加载）过程：它从CSV文件中读取数据，对数据进行映射和转化，然后使用SQL查询在一个临时视图上查找年龄大于30的人，最后将结果输出到某个自定义的Sink上。

#### Virtual Tables（虚拟表）

在环境中注册之后，我们就可以在 SQL 中直接使用这张表进行查询转换了。

```undefined
Table newTable = tableEnv.sqlQuery("SELECT name, age FROM people WHERE age > 30");
```

得到的 newTable 是一个中间转换结果，如果之后又希望直接使用这个表执行 SQL，又该怎么做呢？由于 newTable 是一个 Table 对象，并没有在表环境中注册，所以我们还需要将这个中间结果表注册到环境中，才能在 SQL 中使用：

```undefined
tableEnv.createTemporaryView("NewTable", newTable);
```

这里的注册其实是创建了一个“虚拟表”（Virtual Table）。这个概念与 SQL 语法中的视图（View）非常类似，所以调用的方法也叫作创建“虚拟视图” （createTemporaryView）。

#### 表流互转

```undefined
// 将表转换成数据流，并打印 tableEnv.toDataStream(result).print(); // 将数据流转换成表 // 我们还可以在 fromDataStream()方法中增加参数，用来指定提取哪些属性作为表中的字段名，并可以任意指定位置 Table table = tableEnv.fromDataStream(eventStream, $("timestamp").as("ts"),$("url"));
```

#### 动态表和持续查询

在Flink中，动态表（Dynamic Tables）是一种特殊的表，它可以随时间变化。它们通常用于表示无限流数据，例如事件流或服务器日志。与静态表不同，动态表可以在运行时插入、更新和删除行。

动态表可以像静态的批处理表一样进行查询操作。由于数据在不断变化，因此基于它定义的 SQL 查询也不可能执行一次就得到最终结果。这样一来，我们对动态表的查询也就永远不会停止，一直在随着新数据的到来而继续执行。这样的查询就被称作持续查询（Continuous Query）。

下面是一个简单的例子，它使用Java编写了一个Flink程序，该程序从名为"input-topic"的Kafka主题中读取JSON格式的数据（属性包括"name"和"age"），过滤出所有年龄大于30岁的记录，并将结果输出到另一个名为"output-topic"的Kafka主题中。同时，处理的结果也会在控制台上打印出来。

```undefined
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build(); StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, settings); tableEnv.executeSql("CREATE TABLE input (" + " name STRING," + " age INT" + ") WITH (" + " 'connector' = 'kafka'," + " 'topic' = 'input-topic'," + " 'properties.bootstrap.servers' = 'localhost:9092'," + " 'format' = 'json'" + ")"); tableEnv.executeSql("CREATE TABLE output (" + " name STRING," + " age INT" + ") WITH (" + " 'connector' = 'kafka'," + " 'topic' = 'output-topic'," + " 'properties.bootstrap.servers' = 'localhost:9092'," + " 'format' = 'json'" + ")"); Table result = tableEnv.sqlQuery("SELECT name, age FROM input WHERE age > 30"); tableEnv.toAppendStream(result, Row.class).print(); result.executeInsert("output"); env.execute(); }
```

#### 连接到外部系统

在 Table API编写的 Flink 程序中，可以在创建表的时候用 WITH 子句指定连接器（connector），这样就可以连接到外部系统进行数据交互。

其中最简单的当然就是连接到控制台打印输出：

```undefined
CREATE TABLE ResultTable ( user STRING, cnt BIGINT WITH ( 'connector' = 'print' );
```

##### Kafka

需要导入maven依赖：

```undefined
<dependency> <groupId>org.apache.flink</groupId> <artifactId>flink-connector-kafka_2.12</artifactId> <version>1.13.6</version> </dependency>
```

创建一个连接到 Kafka 的表，需要在 CREATE TABLE 的 DDL 中在 WITH 子句里指定连接器为 Kafka，并定义必要的配置参数：

```sql
CREATE TABLE KafkaTable ( `user` STRING, `url` STRING, `ts` TIMESTAMP(3) METADATA FROM 'timestamp' ) WITH ( 'connector' = 'kafka', 'topic' = 'events', 'properties.bootstrap.servers' = 'localhost:9092', 'properties.group.id' = 'testGroup', 'scan.startup.mode' = 'earliest-offset', 'format' = 'csv' )
```

##### MySQL

```xml
<dependency> <groupId>org.apache.flink</groupId> <artifactId>flink-connector-jdbc_2.12</artifactId> <version>1.13.6</version> </dependency>
```

创建 JDBC 表的方法与前面 Kafka 大同小异：

```sql
-- 创建一张连接到 MySQL 的 表 CREATE TABLE MyTable ( id BIGINT, name STRING, age INT, status BOOLEAN, PRIMARY KEY (id) NOT ENFORCED ) WITH ( 'connector' = 'jdbc', 'url' = 'jdbc:mysql://localhost:3306/mydatabase', 'table-name' = 'users' ); -- 将另一张表 T 的数据写入到 MyTable 表中 INSERT INTO MyTable SELECT id, name, age, status FROM T;
```

### Table API实战

#### 1.创建Table

Table API中已经提供了TableSource从外部系统获取数据，例如常见的数据库、文件系统和Kafka消息队列等外部系统。

1.  从文件中创建Table（静态表）
    
    Flink允许用户从本地或者分布式文件系统中读取和写入数据，只需指定相应的参数即可。但是文件格式必须是CSV格式的。其他文件格式也支持（在Flink中还有Connector等来支持其他格式或者自定义TableSource）
    
    ```java
    public static void main(String[] args) throws Exception { // 创建流式计算的上下文环境 final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); // 创建Table API的上下文环境 StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env); // 创建CSV表源 String sourceDDL = "CREATE TABLE exampleTab (" + "`id` INT, " + "`name` STRING, " + "`score` DOUBLE" + ") WITH (" + "'connector' = 'filesystem'," + "'path' = 'D:\\code\\StudyFlink\\data\\tableexamples'," + "'format' = 'csv'" + ")"; tableEnv.executeSql(sourceDDL); // 打印表结构 ResolvedSchema schema = tableEnv.from("exampleTab").getResolvedSchema(); System.out.println(schema.toString()); }
    ```
    
2.  从DataStream中创建 Table（动态表）
    
    前面已经知道Table API是构建在DataStream API和DataSet API之上的一层更高级的抽象，因此用户可以灵活地使用Table API将Table转换成DataStream或DataSet数据集，也可以将DataSteam或DataSet数据集转换成Table，这和Spark中的DataFrame和RDD的关系类似。
    
    ```java
    public static void main(String[] args) throws Exception { // 先创建StreamExecutionEnvironment final StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment(); EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build(); StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings); // 创建一个DataStream DataStream<Tuple2<String, Integer>> stream = bsEnv.fromElements(Tuple2.of("Alice", 3), Tuple2.of("Bob", 4)); // 将DataStream转化为Table Table table1 = bsTableEnv.fromDataStream(stream); // 再把Table转回DataStream DataStream<Row> streamAgain = bsTableEnv.toDataStream(table1); }
    ```
    

#### 2.查询和过滤

在Table对象上使用`select`操作符查询需要获取的指定字段，也可以使用`filter`或`where`方法过滤字段和检索条件，将需要的数据检索出来。

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment(); streamEnv.setParallelism(1); // Create the Table API execution environment. StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv); SingleOutputStreamOperator<Tuple5<String, String, String, Long, Long>> data = streamEnv.socketTextStream("hadoop101", 8888) .map(new MapFunction<String, Tuple5<String, String, String, Long, Long>>() { @Override public Tuple5<String, String, String, Long, Long> map(String line) throws Exception { String[] arr = line.split(","); return new Tuple5<>(arr[0].trim(), arr[1].trim(), arr[2].trim(), Long.parseLong(arr[4].trim()), Long.parseLong(arr[5].trim())); } }); Table table = tableEnv.fromDataStream(data); // Query tableEnv.toAppendStream(table.select("f0 AS sid, f1 AS type, f3 AS callTime, f4 AS callOut"), Row.class) .print(); // Filter Query tableEnv.toAppendStream(table.filter("f1 === 'success'").where("f1 === 'success'"), Row.class) .print(); tableEnv.execute("sql"); }
```

这段代码从一个指定的socket中读取文本数据，将每一行数据映射为一个5元组（Tuple5），然后把这个数据流转换为表，并进行查询操作。首先，它进行简单的列选择查询并打印结果；然后，它进行筛选查询，选取第二字段"成功"的记录并打印出来。整个过程在一个名为"sql"的任务中执行。

#### 3.UDF自定义函数

用户可以在Table API中自定义函数类，常见的抽象类和接口是：

-   ScalarFunction
-   TableFunction
-   AggregateFunction
-   TableAggregateFunction

```java
public static void main(String[] args) { EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build(); TableEnvironment tableEnv = TableEnvironment.create(settings); // 注册UDF tableEnv.createTemporarySystemFunction("UpperCase", UpperCaseFunction.class); // 使用UDF tableEnv.executeSql( "SELECT UpperCase(myField) FROM myTable" ); } public static class UpperCaseFunction extends ScalarFunction { public String eval(String str) { return str.toUpperCase(); } }
```

这段代码创建了自定义函数（UDF）并使用它。首先，它设置了 Flink 的环境，并通过 Blink Planner 以批处理模式运行。然后，它注册了一个名为 "UpperCase" 的 UDF，该函数将输入字符串转换为大写。最后，它在 SQL 查询中使用了这个 UDF，将 "myTable" 中的 "myField" 字段的值转换成大写形式。

#### 4.Window

```java
public static void main(String[] args) throws Exception { StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env); // 创建一个具有 Process Time 时间属性的表 tableEnv.executeSql( "CREATE TABLE Orders (" + "orderId INT, " + "price DOUBLE, " + "buyer STRING, " + "orderTime TIMESTAMP(3)," + "pt AS PROCTIME()" + // 使用处理时间 ") WITH ('connector' = '...', ...)" ); Table orders = tableEnv.from("Orders"); Table result1 = orders.window(Tumble.over(lit(10).minutes()).on($("pt")).as("w")) .groupBy($("w"), $("buyer")) .select($("buyer"), $("w").start().as("start"), $("w").end().as("end"), $("price").sum().as("totalPrice")); // 创建一个具有 Event Time 时间属性的表，使用Watermarks tableEnv.executeSql( "CREATE TABLE OrdersEventTime (" + "orderId INT, " + "price DOUBLE, " + "buyer STRING, " + "orderTime TIMESTAMP(3), " + "WATERMARK FOR orderTime AS orderTime - INTERVAL '5' SECOND" + // 使用事件时间和水印 ") WITH ('connector' = '...', ...)" ); Table ordersEventTime = tableEnv.from("OrdersEventTime"); Table result2 = ordersEventTime.window(Tumble.over(lit(10).minutes()).on($("orderTime")).as("w")) .groupBy($("w"), $("buyer")) .select($("buyer"), $("w").start().as("start"), $("w").end().as("end"), $("price").sum().as("totalPrice")); // 对于 IngestionTime，Flink 1.12 中已经不推荐使用，因此在 Flink 1.13.6 版本中，你应该使用 ProcessTime 或 EventTime。 }
```

这段代码创建了两个表：一个使用处理时间(Process Time)，另一个使用事件时间(Event Time)并设置了水印。针对这两个表，分别在买家(buyer)和10分钟的时间窗口上进行分组，并计算了每个时间窗口中的总价(totalPrice)。

### 多类型数据流

在 Flink 中，`DataStream`，`ChangelogStream`，`AppendStream`和 `RetractStream` 用于表示不同类型的数据流。简单来说，它们之间的主要区别和联系如下：

-   **DataStream**：这是 Flink 的基础抽象，它表示一个无界的数据流，可以包含任何类型的元素。
-   **toChangelogStream**：这个方法将表转换为一个 ChangeLog 模式的 DataStream。每条记录都代表一个添加、修改或删除的事件。事件通常由可选的元数据标记（例如，'+'（添加）或'-'（撤销））、更新时间以及唯一的键和值组成。ChangelogStream 主要用于处理动态表，并且支持插入，更新和删除操作。
-   **toAppendStream**：这个方法将表转换为一个只包含添加操作的 DataStream。换句话说，结果表只包含插入（append）操作，不能执行更新或删除操作。如果查询的结果表支持删除或更新，则此方法会抛出异常。
-   **toRetractStream**：这个方法将表转换为一个包含添加和撤销消息的 DataStream。每一条添加消息表示在结果表中插入了一行，而每一条撤销消息表示在结果表中删除了一行。如果撤销消息后没有相应的添加消息，那么可能是因为输入数据发生了变化，导致之前发送的结果不再正确，需要被撤销。

### Flink SQL

**企业中Flink SQL比Table API用的多**

Flink SQL 是 Apache Flink 提供的一种使用 SQL 查询和处理数据的方式。它允许用户通过 SQL 语句对数据流或批处理数据进行查询、转换和分析，无需编写复杂的代码。Flink SQL 提供了一种更直观、易于理解和使用的方式来处理数据，同时也可以与 Flink 的其他功能无缝集成。

Flink SQL 支持 ANSI SQL 标准，并提供了许多扩展和优化来适应流式处理和批处理场景。它能够处理无界数据流，具备事件时间和处理时间的语义，支持窗口、聚合、连接等常见的数据操作，还提供了丰富的内置函数和扩展插件机制。

下面是一个简单的 Flink SQL 代码示例，展示了如何使用 Flink SQL 对流式数据进行查询和转换。

```java
public static void main(String[] args) throws Exception { final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.setParallelism(1); // 设置并行度为1，方便观察输出结果 // 创建 Kafka 数据源 Properties properties = new Properties(); properties.setProperty("bootstrap.servers", "localhost:9092"); properties.setProperty("group.id", "flink-consumer"); FlinkKafkaConsumer<String> kafkaConsumer = new FlinkKafkaConsumer<>("input-topic", new SimpleStringSchema(), properties); DataStream<String> sourceStream = env.addSource(kafkaConsumer); // 获取 StreamTableEnvironment StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env); // 注册数据源表 tableEnv.createTemporaryView("source_table", sourceStream, "message"); // 执行 SQL 查询和转换 String query = "SELECT message, COUNT(*) AS count FROM source_table GROUP BY message"; // 执行 SQL 查询和转换 Table resultTable = tableEnv.sqlQuery(query); DataStream<Result> resultStream = tableEnv.toDataStream(resultTable) .map(row -> new Result(row.getField(0).toString(), (Long) row.getField(1))); // 打印结果 resultStream.print(); env.execute("Flink SQL Example"); } // 自定义结果类 public static class Result { public String message; public Long count; public Result() { } public Result(String message, Long count) { this.message = message; this.count = count; } @Override public String toString() { return "Result{" + "message='" + message + '\'' + ", count=" + count + '}'; } }
```

在上述示例中，我们使用 Kafka 作为数据源，并创建了一个消费者从名为 "input-topic" 的 Kafka 主题中读取数据。然后，我们将数据流注册为名为 "source\_table" 的临时表。

接下来，我们使用 Flink SQL 执行 SQL 查询和转换。在这个例子中，我们查询 "source\_table" 表，对 "message" 字段进行分组并计算每个消息出现的次数。查询结果会映射到自定义的 `Result` 类，并最终通过 `print()` 方法打印到标准输出。

最后，我们通过调用 `env.execute()` 方法来启动 Flink 作业的执行。

#### Flink SQL中使用窗口函数

Flink SQL中使用滚动窗口，滑动窗口和会话窗口代码示例如下：

```java
public static void main(String[] args) throws Exception { // 初始化流处理执行环境 final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); final StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env); // 对于实际应用程序，请替换为你的数据源 String sourceDDL = "CREATE TABLE MySourceTable (\n" + " user_id STRING,\n" + " event_time TIMESTAMP(3),\n" + " price DOUBLE\n" + ") WITH (\n" + "'connector' = '...',\n" + "...);\n"; tableEnv.executeSql(sourceDDL); // 滚动窗口 String tumblingWindowQuery = "SELECT user_id, SUM(price) as total_price\n" + "FROM MySourceTable\n" + "GROUP BY user_id, TUMBLE(event_time, INTERVAL '1' HOUR)"; Table tumblingWindowResult = tableEnv.sqlQuery(tumblingWindowQuery); // 滑动窗口 String slidingWindowQuery = "SELECT user_id, SUM(price) as total_price\n" + "FROM MySourceTable\n" + "GROUP BY user_id, HOP(event_time, INTERVAL '30' MINUTE, INTERVAL '1' HOUR)"; Table slidingWindowResult = tableEnv.sqlQuery(slidingWindowQuery); // 会话窗口 String sessionWindowQuery = "SELECT user_id, SUM(price) as total_price\n" + "FROM MySourceTable\n" + "GROUP BY user_id, SESSION(event_time, INTERVAL '1' HOUR)"; Table sessionWindowResult = tableEnv.sqlQuery(sessionWindowQuery); }
```

程序定义了三种不同类型的窗口查询：滚动窗口(tumbling window)，滑动窗口(sliding window)，会话窗口(session window)。

-   滚动窗口：该查询对"MySourceTable"中的数据应用滚动窗口，窗口大小为1小时，并按user\_id进行分组。每个窗口内，会计算每个用户的总价格(sum(price))。
-   滑动窗口：与滚动窗口相似, 但是窗口可以重叠. 这个查询每半小时滑动一次, 并且每次滑动都会创建一个1小时大小的窗口, 再进行与滚动窗口查询相同的计算.
-   会话窗口：会话窗口是根据数据活跃度来划分的，当一个会话内一段时间(这里设定为1小时)没有新的数据到达时，就认为会话结束。该查询按user\_id和event\_time的会话窗口进行分组，然后在每个窗口中计算总价格。

每个查询调用`tableEnv.sqlQuery(query)`方法，并将结果存储在Table对象中。注意这些查询在调用sqlQuery时并没有立即执行，只有当你对结果做出动作（如print、collect或者写入外部系统）时，才会触发执行。

## Flink内存优化

在大数据领域，大多数开源框架（Hadoop、Spark、Flink）都是基于JVM运行，但是JVM的内存管理机制往往存在着诸多类似`OutOfMemoryError`的问题，主要是因为创建过多的对象实例而超过JVM的最大堆内存限制，却没有被有效回收掉。

这在很大程度上影响了系统的稳定性，尤其对于大数据应用，面对大量的数据对象产生，仅仅靠JVM所提供的各种垃圾回收机制很难解决内存溢出的问题。

在开源框架中有很多框架都实现了自己的内存管理，例如Apache Spark的Tungsten项目，在一定程度上减轻了框架对JVM垃圾回收机制的依赖，从而更好地使用JVM来处理大规模数据集。

**Flink也基于JVM实现了自己的内存管理，将JVM根据内存区分为Unmanned Heap、Flink Managed Heap、Network Buffers三个区域**

在Flink内部对Flink Managed Heap进行管理，在启动集群的过程中直接将堆内存初始化成Memory Pages Pool，也就是将内存全部以二进制数组的方式占用，形成虚拟内存使用空间。

新创建的对象都是以序列化成二进制数据的方式存储在内存页面池中，当完成计算后数据对象Flink就会将Page置空，而不是通过JVM进行垃圾回收，保证数据对象的创建永远不会超过JVM堆内存大小，也有效地避免了因为频繁GC导致的系统稳定性问题。

### JobManager配置

JobManager在Flink系统中主要承担管理集群资源、接收任务、调度Task、收集任务状态以及管理TaskManager的功能，JobManager本身并不直接参与数据的计算过程，因此JobManager的内存配置项不是特别多，只要指定JobManager堆内存大小即可。

-   **jobmanager.heap.size**：设定JobManager堆内存大小，默认为1024MB。

### TaskManager配置

TaskManager作为Flink集群中的工作节点，所有任务的计算逻辑均执行在TaskManager之上，因此对TaskManager内存配置显得尤为重要，可以通过以下参数配置对TaskManager进行优化和调整。

-   **taskmanager.heap.size**：设定TaskManager堆内存大小，默认值为1024M，如果在Yarn的集群中，TaskManager取决于Yarn分配给TaskManager Container的内存大小，且Yarn环境下一般会减掉一部分内存用于Container的容错。
    
-   **taskmanager.jvm-exit-on-oom**：设定TaskManager是否会因为JVM发生内存溢出而停止，默认为false，当TaskManager发生内存溢出时，也不会导致TaskManager停止。
    
-   **taskmanager.memory.size**：设定TaskManager内存大小，默认为0，如果不设定该值将会使用`taskmanager.memory.fraction`作为内存分配依据。
    
-   **taskmanager.memory.fraction**：设定TaskManager堆中去除Network Buffers内存后的内存分配比例。该内存主要用于TaskManager任务排序、缓存中间结果等操作。例如，如果设定为0.8，则代表TaskManager保留80%内存用于中间结果数据的缓存，剩下20%的内存用于创建用户定义函数中的数据对象存储。注意，该参数只有在`taskmanager.memory.size`不设定的情况下才生效。
    
-   **taskmanager.memory.off-heap**：设置是否开启堆外内存供Managed Memory或者Network Buffers使用。
    
-   **taskmanager.memory.preallocate**：设置是否在启动TaskManager过程中直接分配TaskManager管理内存。
    
-   **taskmanager.numberOfTaskSlots**：每个TaskManager分配的slot数量。
    

### Flink的网络缓存优化

Flink将JVM堆内存切分为三个部分，其中一部分为Network Buffers内存。Network Buffers内存是Flink数据交互层的关键内存资源，主要目的是缓存分布式数据处理过程中的输入数据。

通常情况下，比较大的Network Buffers意味着更高的吞吐量。如果系统出现“Insufficient number of network buffers”的错误，一般是因为Network Buffers配置过低导致，因此，在这种情况下需要适当调整TaskManager上Network Buffers的内存大小，以使得系统能够达到相对较高的吞吐量。

目前Flink能够调整Network Buffer内存大小的方式有两种：一种是通过直接指定Network Buffers内存数量的方式，另外一种是通过配置内存比例的方式。

#### 设定Network Buffer内存数量（过时）

直接设定Nework Buffer数量需要通过如下公式计算得出：

`NetworkBuffersNum = total-degree-of-parallelism \* intra-node-parallelism * n`

其中`total-degree-of-parallelism`表示每个TaskManager的总并发数量，`intra-node-parallelism`表示每个TaskManager输入数据源的并发数量，n表示在预估计算过程中Repar-titioning或Broadcasting操作并行的数量。`intra-node-parallelism`通常情况下与Task-Manager的所占有的CPU数一致，且Repartitioning和Broadcating一般下不会超过4个并发。可以将计算公式转化如下：

`NetworkBuffersNum = <slots-per-TM>^2 \* < TMs>* 4`

其中slots-per-TM是每个TaskManager上分配的slots数量，TMs是TaskManager的总数量。对于一个含有20个TaskManager，每个TaskManager含有8个Slot的集群来说，总共需要的Network Buffer数量为8^2\*204=5120个，因此集群中配置Network Buffer内存的大小约为160M较为合适。

计算完Network Buffer数量后，可以通过添加如下两个参数对Network Buffer内存进行配置。其中segment-size为每个Network Buffer的内存大小，默认为32KB，一般不需要修改，通过设定numberOfBuffers参数以达到计算出的内存大小要求。

-   **taskmanager.network.numberOfBuffers**：指定Network堆栈Buffer内存块的数量。
    
-   **taskmanager.memory.segment-size**：内存管理器和Network栈使用的内存Buffer大小，默认为32KB。
    

#### 设定Network Buffer内存比例（推荐）

从1.3版本开始，Flink就提供了通过指定内存比例的方式设置Network Buffer内存大小。

-   **taskmanager.network.memory.fraction**：JVM中用于Network Buffers的内存比例。
    
-   **taskmanager.network.memory.min**：最小的Network Buffers内存大小，默认为64MB。
    
-   **taskmanager.network.memory.max**：最大的Network Buffers内存大小，默认1GB。
    
-   **taskmanager.memory.segment-size**：内存管理器和Network栈使用的Buffer大小，默认为32KB。
    

## 结语

感谢你耐心地读到这里，在我们结束这篇博客的同时，我鼓励你继续探索和实践Flink的无尽可能性。无论你是初学者还是专业人士，Flink都有许多值得挖掘的深度和广度。这就像一场数据处理的冒险，充满了挑战与机遇。无论你走到哪一步，都记得享受过程，因为每一个问题的解决都代表着新的认知和成长。

再次感谢你的阅读，希望这篇文章能够带给你收获以及深入的思考，期待你在Flink的学习旅程中取得更大的进步。