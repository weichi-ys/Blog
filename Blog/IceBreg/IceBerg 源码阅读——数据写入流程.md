> 目前数据湖是一个比较火的方向，它确实能解决传统数仓的很多问题，但相关资料还比较少，尤其是 Iceberg。Iceberg 原理介绍的文章网上已经比较多了，但对于实现细节依然是一头雾水，我准备写一些列关于Iceberg 源码分析的文档，在有了充分的掌握后，再总结分析 Iceberg 是如何解决传统数仓遇到的问题。如遇到任何问题，欢迎在公众号 大数据修炼手册 上与我联系。

首先看下面这张图，思考一下如果让我们自己来设计写入流程，我们会怎么设计呢？

![[Picture/e280c9edc9ef6f4ab4db65f98e4ce6f0_MD5.png]]

大体上可以分为以下步骤：

1.  spark 引擎层调用某个接口将数据一条一条往下发，iceberg 接受到数据后, 将数据按照指定的格式写入对应的存储中
2.  在 spark 数据写入完成时，iceberg 按照自己的表规范生成对应的 meta 文件

Iceberg 数据写入会分成两部分来进行分析：

1.  第一部分介绍 数据写入流程，此篇博客的主要内能。
2.  第二部分介绍 数据提交流程，会再下篇博客中进行讲解。

这里数据格式以 parquet 为例，底层存储采用 hdfs, iceberg catalog 使用 hadoop 类型 来分析数据的写入流程。

不会涉及到过多的 spark 知识，只需要知道数据写入发生在 Exector 中，数据提交发生在 Driver 中即可。

## **Spark 引擎层和 Iceberg 对接**

根据 spark 的规范，如果要让 spark 往下游写入数据，需要实现下面几个接口

下面分析的核心逻辑就是如何创建出用来写数据的 `DataWriter`。

为了创建 `DataWriter` 的实现类，先要创建一个类实现spark 中的 `BatchWrite` 接口。先看一下 BatchWrite 实现类是如何创建出来的

1.  在示例代码配置中将 catalog 指定了 iceberg 中定义的 `org.apache.iceberg.spark.SparkCatalog`
2.  在spark sql 解析表时，会通过此 catalog `loadTable` 方法创建出 `SparkTable`
3.  `SparkTable` 中 `newWriteBuilder` 方法创建出 `SparkWriteBuilder`
4.  `SparkWriteBuilder` 实现了spark 中的 `WriteBuilder` 接口，此接口主要作用是用来生成 batch/streaming 对应的 writer。`SparkWriteBuilder` 中 buildForBatch 方法的核心逻辑 1) 生成 SparkWrite 2) 根据不同的写入方式, 通过SparkWrite生成对应的 `BatchWrite(spark 中的写入接口)` 类

```
  // 类 org.apache.iceberg.spark.source.SparkWriteBuilder
  @Override
  public BatchWrite buildForBatch() {
  
    SparkWrite write = new SparkWrite(spark, table, writeInfo, appId, wapId, writeSchema, dsSchema);
    if (overwriteByFilter) {
      return write.asOverwriteByFilter(overwriteExpr);
    } else if (overwriteDynamic) {
      return write.asDynamicOverwrite();
    } else if (overwriteFiles) {
      return write.asCopyOnWriteMergeWrite(mergeScan, isolationLevel);
    } else {
      return write.asBatchAppend();
    }
  }
```

BatchWrite 是我们遇到的第一个比较重要的接口，其定义了 spark 如何将数据写入对应的 datasource 中，这里的datasource 就是指 iceberg。我们近距离看一下的 BatchWrite 定义。

```
// org.apache.spark.sql.connector.write.BatchWrite
public interface BatchWrite {

  DataWriterFactory createBatchWriterFactory(PhysicalWriteInfo info);

  default boolean useCommitCoordinator() {
    return true;
  }

  default void onDataWriterCommit(WriterCommitMessage message) {}

  //数据全部写入完成后进行提交
  void commit(WriterCommitMessage[] messages);

  void abort(WriterCommitMessage[] messages);
}
```

源码上有详细的注释，这里做一个简单的总结，首先创建出一个 DataWriterFactory，DataWriterFactory 为每个分区创建对应的 DataWriter，通过 DataWriter 来向下游写入数据，DataWriter 对数据写入完成后, 调用自己的commit 方法进行提交，如果所有的writer 都提交完毕后，调用BatchWrite 的commit 进行最后的提交。

上述流程不是我们分析的重点，而且也不是直接的简单调用流程，流程图就不画了。

接着我们以 append 模式为例，分析 Iceberg 的写入流程，这篇博客主要分析 数据写入流程，数据提交流程，会放在下篇博客中进行分析。

在 append 模式中 `BaseBatchWrite` 对应的实现类是 `org.apache.iceberg.spark.source.SparkWrite.BatchAppend`, 通过其 `createBatchWriterFactory` 方法创建的类是 `org.apache.iceberg.spark.source.SparkWrite.WriterFactory`，实现了 `DataWriterFactory` 接口。

`WriterFactory` 用来生成 spark 引擎层需要使用的 DataWriter。在 `WriterFactory` 中的 `createWriter` 方法中

1.  创建 OutputFileFactory，用来生成表示底层存储的文件，例如 HadoopOutputFile
2.  创建 `SparkAppenderFactory`，用来生成 FileAppender ,进行数据写入
3.  创建 FileIO , 用来对底层存储进行操作，比如写文件，删除文件，实现类有 `HadoopFileIO`
4.  根据是否是分区表，创建对应的Writer,。我们这里以非分区 `Unpartitioned3Writer` 进行分析

```
//org.apache.iceberg.spark.source.SparkWrite.WriterFactory
public DataWriter<InternalRow> createWriter(int partitionId, long taskId, long epochId) {
      Table table = tableBroadcast.value();

      //根据存储路径 生成表示对应底层的文件 比如 HadoopOutputFile
      OutputFileFactory fileFactory = OutputFileFactory.builderFor(table, partitionId, taskId).format(format).build();
      //
      SparkAppenderFactory appenderFactory = SparkAppenderFactory.builderFor(table, writeSchema, dsSchema).build();
      PartitionSpec spec = table.spec();
      //包含了数据的写入路径
      FileIO io = table.io();

      if (spec.isUnpartitioned()) {
        return new Unpartitioned3Writer(spec, format, appenderFactory, fileFactory, io, targetFileSize);
      } else if (partitionedFanoutEnabled) {
        return new PartitionedFanout3Writer(
            spec, format, appenderFactory, fileFactory, io, targetFileSize, writeSchema, dsSchema);
      } else {
        return new Partitioned3Writer(
            spec, format, appenderFactory, fileFactory, io, targetFileSize, writeSchema, dsSchema);
      }
    }
  }
```

经过上面的一系列操作，终于生成了 spark 引擎层用来往 Iceberg 写数据的 DataWriter 的实现类。

先看一下 DataWriter 接口的定义，此接口定义了spark 引擎层如何将数据一条一条往下游写，在写入完成之后便可以进行 commit。详细信息可以查看源码上的注释。

其中 write 方法中的参数 record ，表示 spark 引擎层要写入的一条数据，一般是 spark 中的 `InternalRow` 类型。接下来的代码逻辑将转入到 iceberg 中。

```
public interface DataWriter<T> extends Closeable {

  void write(T record) throws IOException;
 
  WriterCommitMessage commit() throws IOException;

  void abort() throws IOException;
}
```

先用一张流程图总结一下数据写入涉及的类。

![[Picture/6d0c8344813388e6eb5d039215944409_MD5.png]]

`Unpartitioned3Writer` 实现了 spark 的 `DataWriter` 接口，只是简单的将 write 委托给了 `RollingFileWriter`

```
public class UnpartitionedWriter<T> extends BaseTaskWriter<T> {

  private final RollingFileWriter currentWriter;

  public UnpartitionedWriter(PartitionSpec spec, FileFormat format, FileAppenderFactory<T> appenderFactory,
                             OutputFileFactory fileFactory, FileIO io, long targetFileSize) {
    super(spec, format, appenderFactory, fileFactory, io, targetFileSize);
    currentWriter = new RollingFileWriter(null);
  }

  @Override
  public void write(T record) throws IOException {
    currentWriter.write(record);
  }

  @Override
  public void close() throws IOException {
    currentWriter.close();
  }
}
```

`RollingFileWriter` 继承自 `BaseRollingWriter` 如下图所示

![[Picture/350f1bfaf88aebba86605609528c4bb7_MD5.png]]

`BaseRollingWriter` 实现了 write(T record) 方法，由名字可知，如果数据将一个文件写满后，会结束当前文件，开启一个新文件继续进行写入。所以 `BaseRollingWriter`定义了写入模板，首先在构造方法中打开一个当前文件，然后 write 时判断当前文件是否写满了，如果写满了对当前文件进行close，开启一个新文件继续写入，而对应的写入方法交给子类来实现。

`RollingFileWriter` 首先通过 SparkAppenderFactory 的newDataWriter方法 创建出一个DataWriter, DataWriter 实际将写入操作委托给了 FileAppender ，如果parquet 格式，对应的实现类是 ParquetWriter 。这一步是将 iceberg的写入转交给了对应的 Format 格式来进行写入。

```
//org.apache.iceberg.spark.source.SparkAppenderFactory
@Override
  public DataWriter<InternalRow> newDataWriter(EncryptedOutputFile file, FileFormat format, StructLike partition) {
    return new DataWriter<>(newAppender(file.encryptingOutputFile(), format), format,
        file.encryptingOutputFile().location(), spec, partition, file.keyMetadata());
  }

  @Override
  // 最后生成了 ParquetWriter，按照parquert 格式对数据进行写入
  public FileAppender<InternalRow> newAppender(OutputFile file, FileFormat fileFormat) {
      switch (fileFormat) {
        case PARQUET:
          return Parquet.write(file)
              .createWriterFunc(msgType -> SparkParquetWriters.buildWriter(dsSchema, msgType))
              .setAll(properties)
              .metricsConfig(metricsConfig)
              .schema(writeSchema)
              .overwrite()
              .build();
}
}
```

### **写入Parquet 文件**

`DataWriter` 将数据的写入委托给了 FileAppender 的子类`ParquetWriter`, `ParquetWriter` 负责生成parquet 格式文件，并最终写入到对应的底层存储中，如 hdfs。

FileAppender 的继承结果如下图所示

![[Picture/65e4f11d2e4666d2e0b7e6fd40c39338_MD5.png]]

在分析 `ParquetWriter` 之前，让我们分析一下`ParquetWriter` 中的 model 变量。model 对应的类型是 ParquetValueWriter，相关的继承结构如下图所示(为了展示方便，删除了部分子类)

model 是在 SparkParquetWriters.buildWriter 使用访问者模式生成的，根据 table schema 的 StructType 中每一列的数据类型生成对应 ParquetValueWriter。`ParquetValueWriter` 是 Iceberg 对 parquet 数据写入的抽象，核心功能是对上游传下来的一行数据，进行拆解，写入列存。其中子类 StructWriter 表示复合类型，比如 `InternalRowWriter` 用来表示一行记录。StructType 中每一列的字段类型都有对应的写入器，比如`PrimitiveWriter`。

![[Picture/f034e894172c2df5f3f19769406f5d95_MD5.png]]

接下来重点看一下 InternalRowWriter 和 StructWriter 的实现。

在SparkParquetWriters.buildWriter 使用访问者模式 创建 ParquetValueWriter 时，会生成每一列数据类型对应的 Writer 传递给 InternalRowWriter 的构造方法，这样 InternalRowWriter 就拥有了每一列对应的 ParquetValueWriter，在对数据进行写入时通过 InternalRowWriter 便可以对 spark 传递下来的一行数据拆解成 列存，写入 parquet 文件中。

```
private static class InternalRowWriter extends ParquetValueWriters.StructWriter<InternalRow> {
    private final DataType[] types;

    private InternalRowWriter(List<ParquetValueWriter<?>> writers, List<DataType> types) {
      super(writers);
      this.types = types.toArray(new DataType[types.size()]);
    }

    // 从row 中提取对应列的数据
    @Override
    protected Object get(InternalRow struct, int index) {
      return struct.get(index, types[index]);
    }
  }

public abstract static class StructWriter<S> implements ParquetValueWriter<S> {
    private final ParquetValueWriter<Object>[] writers;
    private final List<TripleWriter<?>> children;

    @SuppressWarnings("unchecked")
    protected StructWriter(List<ParquetValueWriter<?>> writers) {
      this.writers = (ParquetValueWriter<Object>[]) Array.newInstance(
          ParquetValueWriter.class, writers.size());

      ImmutableList.Builder<TripleWriter<?>> columnsBuilder = ImmutableList.builder();
      for (int i = 0; i < writers.size(); i += 1) {
        ParquetValueWriter<?> writer = writers.get(i);
        this.writers[i] = (ParquetValueWriter<Object>) writer;
        columnsBuilder.addAll(writer.columns());
      }

      this.children = columnsBuilder.build();
    }

    @Override
    public void write(int repetitionLevel, S value) {
      for (int i = 0; i < writers.length; i += 1) {
        Object fieldValue = get(value, i); //提取对应列的数据
        writers[i].write(repetitionLevel, fieldValue); //使用对应数据类型的 Writer 进行写入
      }
    }

    @Override
    public void setColumnStore(ColumnWriteStore columnStore) {
      for (ParquetValueWriter<?> writer : writers) {
        writer.setColumnStore(columnStore);
      }
    }

    protected abstract Object get(S struct, int index);
  }
```

`ParquetWriter` 涉及到很多 Parquet 相关的概念，这里先挖个坑，后面会单独写一篇 Parquet 写入流程分析的代文章。这里先简单讲一下，Parquet 文件写入分为两部 1) 先将数据按列拆分，写入到内存中 对应的类是 ColumnWriteStore ， 2) 将内存数的数据 flush 到底层存储，对应的类是 ParquetFileWriter。

现在再回头对 `ParquetWriter` 分析就比较简单了。

`ParquetWriter` 中的核心方法是 add, 参数 value 代表从 spark 中传递下来的一条记录，依然是 `InternalRow` 类型。所以需要借助 InternalRowWriter 从 InternalRow 中提取每一列字段对应的值，往列存储中进行写入。

在 startRowGroup 方法中我们会将 ColumnWriteStore 传递给每个 ParquetValueWriter，这样数据就按列写入到内存中。

在 checkSize 时会定期调用 flushRowGroup ，将内存中的数据 flush 到磁盘中。

```

package org.apache.iceberg.parquet;

class ParquetWriter<T> implements FileAppender<T>, Closeable {

  private final ParquetValueWriter<T> model;
  private final ParquetFileWriter writer;
  private ColumnWriteStore writeStore;
  
  @SuppressWarnings("unchecked")
  ParquetWriter(Configuration conf, OutputFile output, Schema schema, long rowGroupSize,
                Map<String, String> metadata,
                Function<MessageType, ParquetValueWriter<?>> createWriterFunc,
                CompressionCodecName codec,
                ParquetProperties properties,
                MetricsConfig metricsConfig,
                ParquetFileWriter.Mode writeMode) {
    
    this.model = (ParquetValueWriter<T>) createWriterFunc.apply(parquetSchema);
   
    this.writer = new ParquetFileWriter(ParquetIO.file(output, conf), parquetSchema,
         writeMode, rowGroupSize, 0);
    writer.start();
    startRowGroup();
  }

  @Override
  public void add(T value) {
    recordCount += 1;
    model.write(0, value);
    writeStore.endRecord();
    checkSize();
  }

  private void checkSize() {
    if (recordCount >= nextCheckRecordCount) {
      long bufferedSize = writeStore.getBufferedSize();
      double avgRecordSize = ((double) bufferedSize) / recordCount;

      if (bufferedSize > (nextRowGroupSize - 2 * avgRecordSize)) {
        flushRowGroup(false);
      } else {
        long remainingSpace = nextRowGroupSize - bufferedSize;
        long remainingRecords = (long) (remainingSpace / avgRecordSize);
        this.nextCheckRecordCount = recordCount + Math.min(Math.max(remainingRecords / 2, 100), 10000);
      }
    }
  }

  private void flushRowGroup(boolean finished) {
    try {
      if (recordCount > 0) {
        writer.startBlock(recordCount);
        writeStore.flush();
        flushPageStoreToWriter.invoke(writer);
        writer.endBlock();
        if (!finished) {
          startRowGroup();
        }
      }
    } catch (IOException e) {
      throw new RuntimeIOException(e, "Failed to flush row group");
    }
  }

  private void startRowGroup() {
    try {
      this.nextRowGroupSize = Math.min(writer.getNextRowGroupSize(), targetRowGroupSize);
    } catch (IOException e) {
      throw new RuntimeIOException(e);
    }
    this.nextCheckRecordCount = Math.min(Math.max(recordCount / 2, 100), 10000);
    this.recordCount = 0;

    PageWriteStore pageStore = pageStoreCtorParquet.newInstance(
        compressor, parquetSchema, props.getAllocator(), this.columnIndexTruncateLength);

    this.flushPageStoreToWriter = flushToWriter.bind(pageStore);
    this.writeStore = props.newColumnWriteStore(parquetSchema, pageStore);

    model.setColumnStore(writeStore);
  }

  @Override
  public void close() throws IOException {
    if (!closed) {
      this.closed = true;
      flushRowGroup(true);
      writeStore.close();
      writer.end(metadata);
    }
  }
}
```

在数据写入到文件后，iceberg 需要将此文件的信息记录在metadata 中，此过程我们采用从后往前进行分析。

DataWriter 进行close时，会先将 appender 进行close, 然后生成 dataFile，表示数据文件的相关信息，其中重要的有 数据格式，存储路径，文件大小，分区信息，统计信息

```
//org.apache.iceberg.io.DataWriter
@Override
  public void close() throws IOException {
    if (dataFile == null) {
      appender.close();
      this.dataFile = DataFiles.builder(spec)
          .withFormat(format)
          .withPath(location)
          .withPartition(partition)
          .withEncryptionKeyMetadata(keyMetadata)
          .withFileSizeInBytes(appender.length())
          .withMetrics(appender.metrics())
          .withSplitOffsets(appender.splitOffsets())
          .withSortOrder(sortOrder)
          .build();
    }
  }
```

当 `BaseRollingWriter` 需要滚动一个文件 或者 调用其close 方法时，会执行closeCurrent 方法，其会先对下游进行close, 然后调用 complete 将下游生成的 DataFile 文件，放在`BaseTaskWriter` 的`completedDataFiles` 列表中。代码示例如下

```
private abstract class BaseRollingWriter<W extends Closeable> implements Closeable {
  private void closeCurrent() throws IOException {
      if (currentWriter != null) {
        currentWriter.close();

        if (currentRows == 0L) {
          io.deleteFile(currentFile.encryptingOutputFile());
        } else {
          complete(currentWriter);
        }
        this.currentRows = 0;
      }
    }
}
```

`Unpartitioned3Writer` 实现了`DataWriter` 接口，需要实现 commit 方法。commit 方法先会对下游进行close.

然后生成 TaskCommit，只能包含数据文件，不能包含 deleteFiles。

```
private static class Unpartitioned3Writer extends UnpartitionedWriter<InternalRow>
      implements DataWriter<InternalRow> {

  @Override
    public WriterCommitMessage commit() throws IOException {
      this.close();
      return new TaskCommit(dataFiles());
    }
}

public interface TaskWriter<T> extends Closeable {
  default DataFile[] dataFiles() throws IOException {
    WriteResult result = complete();
    Preconditions.checkArgument(result.deleteFiles() == null || result.deleteFiles().length == 0,
        "Should have no delete files in this write result.");

    return result.dataFiles();
  }
}

public abstract class BaseTaskWriter<T> implements TaskWriter<T> {
  @Override
  public WriteResult complete() throws IOException {
    close();

    return WriteResult.builder()
        .addDataFiles(completedDataFiles)
        .addDeleteFiles(completedDeleteFiles)
        .addReferencedDataFiles(referencedDataFiles)
        .build();
  }
}
```

至此已经获取到文件写入的信息，等待所有 Exector 写入结束，iceberg 便可提交元数据，iceberg 元数据的提交我们在下篇博客中进行分析。