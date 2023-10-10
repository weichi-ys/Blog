这篇博客，我们分析常规的数据读取流程，不涉及到数据更新，删除等场景下的读取。

数据读取大概可以分为两个步骤

1.  通过 Iceberg 的元数据 snapshot, manifest file 等解析出包含数据文件信息的 DataFile 对象
2.  读取数据文件内容，把每行数据封装成 Spark 的 InternalRow 返回给引擎层

![[Picture/d480b1ac66f0a6aceff2cd748ea29ade_MD5.jpg]]

根据 Spark 规范，如果要让 Spark 读取数据，需要实现以下几个接口。

| Spark 接口 | Iceberg 实现类 | 描述 |
| --- | --- | --- |
| org.apache.spark.sql.connector.catalog.SupportsRead | org.apache.iceberg.spark.source.SparkTable | 生成读取操作的ScanBuilder |
| org.apache.spark.sql.connector.read.ScanBuilder | org.apache.iceberg.spark.source.SparkScanBuilder | 创建表示读表的操作 Scan |
| org.apache.spark.sql.connector.read.Scan | org.apache.iceberg.spark.source.SparkBatchQueryScan | 生成 streaming/batch 对应的读操作 |
| org.apache.spark.sql.connector.read.Batch | org.apache.iceberg.spark.source.SparkBatchQueryScan | 读数据文件进行划分，然后生成数据文件的 reader |
|  |  |  |
| org.apache.spark.sql.connector.read.PartitionReaderFactory | org.apache.iceberg.spark.source.ReaderFactory | 创建出 Reader |
| org.apache.spark.sql.connector.read.PartitionReader | org.apache.spark.sql.connector.read.RowReader | 数据读取迭代器，返回的类型是 InternalRow |

Spark 在读取数据之前，需要为每个 Executor 分配数据文件，然后通过 Reader 读取数据文件。这两个接口都是在 `org.apache.spark.sql.connector.read.Batch` 实现的，创建 Batch 的步骤如下
1.  Iceberg 的 SparkTable 实现了 `SupportsRead` 的 `newScanBuilder` 方法，创建出 `SparkScanBuilder`
2.  `SparkScanBuilder` 会创建出 `SparkBatchQueryScan`
3.  `SparkBatchQueryScan` 的 toBatch 方法创建出表示 批操作 的Batch 对象
4.  通过 `SparkBatchQueryScan` 的 planInputPartitions 获取要读取的数据分片
5.  通过 `SparkBatchQueryScan` , 生成 Reader 读取数据

重点看一下 Batch 接口的定义

```
//org.apache.spark.sql.connector.read.Batch;
public interface Batch {

  //表示一个输入分片
  InputPartition[] planInputPartitions();

  //为每个输入的文件创建 Reader
  PartitionReaderFactory createReaderFactory();
}
```

## 生成数据分片

先通过流程图，看一下涉及到的类

![[Picture/3e3a8f39157d101be1d884aa4612eb78_MD5.jpg]]

Iceberg 中的 `SparkBatchQueryScan` 同时实现了 Spark `Scan` 和 Batch 接口

![[Picture/afd3b9f92fefa7ba7902126d75fbe4e9_MD5.jpg]]

首先 Spark 引擎会调用 `SparkBatchQueryScan` 的 `planInputPartitions` 方法, 获取输入分片。

`SparkBatchQueryScan` 表示普通的查询，SparkMergeScan 用在需要数据合并的场景下。

`planInputPartitions`方法 先调用的 tasks() 方法获取到 CombinedScanTask，然后再封装成 ReadTask 返回给引擎。

ReadTask 实现了 InputPartition接口，但 InputPartition 接口没有定义有用的方法，具体封装什么数据由 ReadTask 决定。

ReadTask 实际上封装的是 每个数据文件的 元信息，最终作为 Spark Reader 的输入。

```
//org.apache.iceberg.spark.source.SparkBatchScan
 @Override
  public InputPartition[] planInputPartitions() {
//生成 CombinedScanTask
    List<CombinedScanTask> scanTasks = tasks(); //task 由子类来实现

    InputPartition[] readTasks = new InputPartition[scanTasks.size()];

    //将 CombinedScanTask 封装成 ReadTask
    Tasks.range(readTasks.length)
        .stopOnFailure()
        .executeWith(localityPreferred ? ThreadPools.getWorkerPool() : null)
        .run(index -> readTasks[index] = new ReadTask(
            scanTasks.get(index), tableBroadcast, expectedSchemaString,
            caseSensitive, localityPreferred));

    return readTasks;
  }
```

在进行数据文件分片之前，已经由表名通过 Catalog 加载了表的 metadata.json 文件，生成BaseTable。

通过 BaseTable 创建出 TableScan,对应的实现类是 `DataTableScan`, 同时指定了一些过滤条件，snapshot 的时间范围等，通过 TableScan 的子类来查找数据文件。

```
protected List<CombinedScanTask> tasks() {
TableScan scan = table() // table() 返回的是BaseTable,参考元数据博客
          .newScan()
          .caseSensitive(caseSensitive())
          .project(expectedSchema());

  //如果单个文件大小超过 SPLIT_SIZE，默认128M,并且支持切分，会对文件进行切分
if (splitSize != null) {
     scan = scan.option(TableProperties.SPLIT_SIZE, splitSize.toString());
  }

//如果文件比较小，比如只有1KB, 会将多个文件打包成一个输入分片，如果文件大小小于 SPLIT_OPEN_FILE_COST 默认4M,会按照 SPLIT_OPEN_FILE_COST 来计算
if (splitOpenFileCost != null) {
        scan = scan.option(TableProperties.SPLIT_OPEN_FILE_COST, splitOpenFileCost.toString());
      }

//如果查询有where 过滤条件，会进行下推，进行文件级别的裁剪
  for (Expression filter : filterExpressions()) {
     scan = scan.filter(filter);
  }

CloseableIterable<CombinedScanTask> tasksIterable = scan.planTasks());
return Lists.newArrayList(tasksIterable)

}
```

TableScan的继承结构如下，通过继承关系，也可以发现 Iceberg 是支持增量读取的

![[Picture/ad1daa855d26a665cc15c100b304b2a9_MD5.jpg]]

`DataTableScan` 通过以下流程来生成输入分片

1.  由 planFiles() 获取所有需要读取的数据文件, 实际是委托给了 ManifestGroup 来操作
2.  对大文件进行拆分，如果单个文件大小超过 SPLIT\_SIZE，默认128M,并且文件格式支持切分，会对文件进行切分
3.  将小文件打包在一起，如果文件比较小，比如只有1KB, 会将多个文件打包成一个输入分片，如果文件大小小于 SPLIT\_OPEN\_FILE\_COST， 默认4M,会按照 SPLIT\_OPEN\_FILE\_COST 来计算

```
//org.apache.iceberg.BaseTableScan
public CloseableIterable<CombinedScanTask> planTasks() {
  //获取到所有数据文件
CloseableIterable<FileScanTask> fileScanTasks = planFiles();
//对大文件进行拆分
  CloseableIterable<FileScanTask> splitFiles = TableScanUtil.splitFiles(fileScanTasks, splitSize);
//把多个小文件合并成在一个 CombinedScanTask 中
  return TableScanUtil.planTasks(splitFiles, splitSize, lookback, openFileCost);
}

public CloseableIterable<FileScanTask> planFiles() {
//先确定好要读取的 snapshot
Snapshot snapshot = snapshot();
return planFiles(ops, snapshot,
          context.rowFilter(), context.ignoreResiduals(), context.caseSensitive(), context.returnColumnStats());
}
```

FileScanTask 的继承关系如下，

FileScanTask 表示一个输入文件或者一个文件的一部分，在这里实现类是 BaseFileScanTask

CombinedScanTask 表示多个 FileScanTask 组合在一起，实现类是 `BaseCombinedScanTask`

![[Picture/6f69f25a6d46ec6a41e4baa1d268b04d_MD5.jpg]]

由于定位文件需要 Manifest 等信息，先通过 snapshot.dataManifests() 读取当前 snapshot 的 manifestlist 文件,

解析出表示 ManifestFile 的对象 。

再构造出一个 ManifestGroup ,让 ManifestGroup 根据 manifest file 来获取输入的文件

```
//org.apache.iceberg.ManifestGroup
@Override
  public CloseableIterable<FileScanTask> planFiles(TableOperations ops, Snapshot snapshot,
                                                   Expression rowFilter, boolean ignoreResiduals,
                                                   boolean caseSensitive, boolean colStats) {
    //此时会通过 snapshot.dataManifests() 读取当前 snapshot 的 manifestlist 文件,解析出表示 ManifestFile 的对象 
    ManifestGroup manifestGroup = new ManifestGroup(ops.io(), snapshot.dataManifests(), snapshot.deleteManifests())
        .caseSensitive(caseSensitive)
        .select(colStats ? SCAN_WITH_STATS_COLUMNS : SCAN_COLUMNS)
        .filterData(rowFilter)
        .specsById(ops.current().specsById())
        .ignoreDeleted();

    if (ignoreResiduals) {
      manifestGroup = manifestGroup.ignoreResiduals();
    }
    return manifestGroup.planFiles();
  }
```

由于不考虑删除等场景，所以获取 文件信息 的流程比较简单，

1.  通过 ManifestFile 对象 去读取所有的 ManifestFile 文件
2.  通过 ManifestFile 解析出 DataFile 对象
3.  如果有谓词下推，会对 DataFile 做过滤，进行文件级别裁剪
4.  将符合条件的 DataFile 封装到 BaseFileScanTask 中

```
public CloseableIterable<FileScanTask> planFiles() {

//将返回的 DataFile 对象, 封装成 BaseFileScanTask
Iterable<CloseableIterable<FileScanTask>> tasks = entries((manifest, entries) -> {
return CloseableIterable.transform(entries, e -> new BaseFileScanTask(
            e.file().copy(), deleteFiles.forEntry(e), schemaString, specString, residuals));
return CloseableIterable.concat(tasks);
}

private <T> Iterable<CloseableIterable<T>> entries(
      BiFunction<ManifestFile, CloseableIterable<ManifestEntry<DataFile>>, CloseableIterable<T>> entryFn) {

  //先通过表达是过滤 ManifestFile 文件
Iterable<ManifestFile> matchingManifests = 
        Iterables.filter(dataManifests, manifest -> evalCache.get(manifest.partitionSpecId()).eval(manifest));

return Iterables.transform(
        matchingManifests,
        manifest -> {
//读取 ManifestFile
          ManifestReader<DataFile> reader = ManifestFiles.read(manifest, io, specsById)
              .filterRows(dataFilter)
              .filterPartitions(partitionFilter)
              .caseSensitive(caseSensitive)
              .select(columns);

//解析出 DataFile 对象
          CloseableIterable<ManifestEntry<DataFile>> entries = reader.entries();
       
//对 DataFile 做裁剪
          if (evaluator != null) {
            entries = CloseableIterable.filter(entries,
                entry -> evaluator.eval((GenericDataFile) entry.file()));
          }
          return entryFn.apply(manifest, entries);
        });
}
```

通过一系列操作，我们获取到了包含数据文件信息的 DataFile 对象，将其封装在 CombinedScanTask 进行返回

## 数据读取

在获取到要读取的数据文件信息后，Spark 会为每个任务分配数据分片，由 Executor 进行读取。

先看一下 Spark 定义的接口 `org.apache.spark.sql.connector.read.PartitionReader` ，很符合火山模型的定义，但 Spark 现在也支持向量化读取。

RowDataReader 实现了`PartitionReader` 接口 ，看一下 RowDataReader 继承关系。

整个数据读取的核心逻辑 就是读取 Parquet 中的数据。

```
//org.apache.spark.sql.connector.read.PartitionReader
public interface PartitionReader<T> extends Closeable {

  /**
   * Proceed to next record, returns false if there is no more records.
   */
  boolean next() throws IOException;

  /**
   * Return the current record. This method should 
return same value until `next` is called.
   */
  T get();
}
```

![[Picture/dc003e1588c0bd2bf678c57da408b9b3_MD5.jpg]]

通过 ReaderFactory 创建出 PartitionReader，比较简单，需要注意一下 RowReader 的输入是 ReadTask，可以把ReadTask 理解成 Iceberg DataFile 对象的封装

```
static class ReaderFactory implements PartitionReaderFactory {

    @Override
    public PartitionReader<InternalRow> createReader(InputPartition partition) {
      if (partition instanceof ReadTask) {
        return new RowReader((ReadTask) partition);
      } 
    }

    @Override  //可以看到 Spark 也是支持向量化读取的
    public PartitionReader<ColumnarBatch> createColumnarReader(InputPartition partition) {
      if (partition instanceof ReadTask) {
        return new BatchReader((ReadTask) partition, batchSize);
      } 
    }
  }
```

BaseDataReader 实现了`PartitionReader` 的 next 接口，之前讲过 CombinedScanTask 里面包含了多个文件，next 把具体的读操作再委托读每个文件

```
// 代码有简化
abstract class BaseDataReader<T> implements Closeable {
private T current = null;
private final Map<String, InputFile> inputFiles;
private CloseableIterator<T> currentIterator;

  //代码有简化，去掉了加密逻辑，用输入分片的所有文件，生成一个 location 和 InputFile, FileIO 表示存储
BaseDataReader(CombinedScanTask task, FileIO io, EncryptionManager encryptionManager) {
    this.tasks = task.files().iterator();
    this.inputFiles files = Maps.newHashMapWithExpectedSize(task.files().size());

task.files().forEach(file -> files.putIfAbsent(file.location(), io.newInputFile(file.location())));
    this.currentIterator = CloseableIterator.empty();
  }

//next 委托给每个文件对应的 Iterator
public boolean next() throws IOException {
      while (true) {
        if (currentIterator.hasNext()) {
          this.current = currentIterator.next();
          return true;
        } else if (tasks.hasNext()) {
          this.currentIterator.close();
          this.currentTask = tasks.next();
          this.currentIterator = open(currentTask); // 读文件
        } else {
          this.currentIterator.close();
          return false;
        }
      }
  }

public T get() {
    return current;
  }

}
```

RowDataReader 会根据文件格式，使用对应的 Format Reader , 通过 newParquetIterable() 方法，这里返回的是 `ParquetFileReade`。

实际读取操作由Parquet 库提供的 `org.apache.parquet.hadoop.ParquetFileReader` 实现

```
//org.apache.iceberg.spark.source.RowDataReader
protected CloseableIterable<InternalRow> open(FileScanTask task, Schema readSchema,Map idToConstant) {
    CloseableIterable<InternalRow> iter;
      InputFile location = getInputFile(task);
      switch (task.file().format()) {
        case PARQUET:
          iter = newParquetIterable(location, task, readSchema, idToConstant);
          break;
      }
    }
    return iter;
  }

private CloseableIterable<InternalRow> newParquetIterable(
      InputFile location,
      FileScanTask task,
      Schema readSchema,
      Map<Integer, ?> idToConstant) {
    Parquet.ReadBuilder builder = Parquet.read(location)
        .reuseContainers()
        .split(task.start(), task.length())
        .project(readSchema)
        .createReaderFunc(fileSchema -> SparkParquetReaders.buildReader(readSchema, fileSchema, idToConstant))
        .filter(task.residual())
        .caseSensitive(caseSensitive);

    return builder.build(); //实际读取操作由Parquet 库提供的 ParquetFileReader 实现
  }
```

至此 数据读取的逻辑就分析完毕，核心就是去读文件，把每行数据封装成 Spark 的 InternalRow，返回给 Spark 引擎。