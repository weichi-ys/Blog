> 目前数据湖是一个比较火的方向，它确实能解决传统数仓的很多问题，但相关资料还比较少，尤其是 Iceberg。Iceberg 原理介绍的文章网上已经比较多了，但对于实现细节依然是一头雾水，我准备写一些列关于Iceberg 源码分析的文档，在有了充分的掌握后，再总结分析 Iceberg 是如何解决传统数仓遇到的问题。如遇到任何问题，欢迎在公众号 大数据修炼手册 上与我联系。  

书接上文，在一篇博客中我们分析了文件的写入流程，这篇分析文件写入后 manifest 等文件的生成和表的提交流程

![[Picture/f965d00a2214118773f3eb42ca78c5bf_MD5.jpg]]

还是看着图来分析，根据架构图在数据文件写入完成后，依次进行以下流程

1.  生成 manifestfile，包含多个数据文件的信息
2.  生成 manifestlist 文件，包含多个manifestfile 文件
3.  生成meta.json 文件，保存了历史所有 snapshot, 每个snapshot 指向一个 manifestlist 文件
4.  向 catalog 提交 snapshot

我们还是从下往上分析基本概念，然后再将提交表的流程

## 基本概念

### manifestfile 文件

在上文中我们提到，在 Exector 文件写入完成后，会为每个文件生成一个 DataFile 对象，主要包含 数据文件的统计信息和 数据文件路径，然后将 DataFile 对象传递给 Driver 进行提交。

这里的实现类主要是 `GenericDataFile` ，使用 avro 格式保留一些基本信息，DeleteFile 我们放在 Flink 篇章进行分析。

![[Picture/8ecb983358393175ba5d1b41a3ed55b7_MD5.jpg]]

manifestfile 文件需要写入到存储上，所以需要一个ManifestWriter, 继承自 FileAppender，manifestfile 文件是 avro 格式的，实际上 ManifestWriter 将写入操作转交给了 AvroFileAppender 。

ManifestWriter 只是将多个 DataFile 对象的数据序列化到磁盘上, 同时在文件中也保留了当前的 snapshot

FileAppender 的写入流程我们在数据写入流程那篇博客，已经分析过了 ParquetWriter的写入，AvroFileAppender 流程类似，我们就不再分析。

![[Picture/24412de172d8f32fe632120f8bef7041_MD5.jpg]]

ManifestWriter 写入结束后，在程序中用 `GenericManifestFile` 表示生成的一个 ManifestFile 文件，只是保存了 ManifestFile 的存储路径 和 ManifestFile由几个文件组成等基本信息。

我们总结一下 ManifestFile 文件的生成，就是通过 ManifestWriter 将 DataFile 对象的数据序列化到磁盘上，然后再程序中用一个 `GenericManifestFile` 对象表示当前生成的 ManifestFile。

### manifestlist

manifestlist 也需要写入到存储上，所以也有一个 ManifestListWriter。

在 append 模式下除了本次新生成的 `GenericManifestFile` 还需要和当前已经持有的 `GenericManifestFile` 做一个合并。

ManifestListWriter 将这些 `GenericManifestFile` 对象以 Avro 格式序列化到 存储上，便生成了 manifestlist 文件。最后生成一个 manifestlist 文件的存储路径

### Snapshot

snapshot 表示一个快照，每次提交生成一个快照。

manifestlist 文件的存储路径 + 新生成的 snapshot id + 上一次的 snapshot id 等，便组成了一个 snapshot。

向 `TableMetadata` 对象中添加本次的 snapshot ，把 current\_snapshot 字段设置为新生成的 snapshot id ，然后通过 tableOperations 向 catalog 做一次提交，表的提交流程就结束了。

tableOperations 的提交流程在元数据那篇博客中已经分析过了，可以简单理解为，1) 生成 matadata.json 文件，同时存储了所有历史的 snapshot 对象，

metadata.json 中的 current\_snapshot 指向最新的一次 snapshot。2) 把catalog 中的 metadata\_location 更新为 metadata.json 文件的存储路径

## 表提交流程

我们还是以 append 模式来讲解整个流程。

为了表示表的一次变更，Iceberg 抽象出了 PendingUpdate 接口，主要接口和继承关系如下, 表示 Append 模式的 MergeAppend，从名字也可以看出会对历史的 Append 进行 merge

```
public interface PendingUpdate<T> {

  //执行更新但不提交
  T apply();

  //向catalog 提交表
  void commit();

  //如果metadata 发生了更新，发送时间，这个很重要，由此我们可以监听到表的表更消息
  default Object updateEvent() {
    return null;
  }
}
```

![[Picture/58609a8a6881d5b31808dc21bab733de_MD5.jpg]]

在数据写入分析流程中我们提到过，spark 定义了一个 BaseBatchWrite 接口，所有executor 本地写入完成后，会向 Driver 发送 `WriterCommitMessage` 消息，当 Driver 汇总了所有的 `WriterCommitMessage` 之后会进行最后的提交。`WriterCommitMessage` 的内容其实就是 Iceberg 的 DataFile 对象。

![[Picture/0270ca5a85dcbb52486bd5d5d75fd367_MD5.jpg]]

BatchAppend 会通过 Table 创建出一个 MergeAppend, 然后把新生成的 DataFile对象 添加到 MergeAppend 中，之后就调用 SnapshotProducer 的 commit() 方法

```
private class BatchAppend extends BaseBatchWrite {
    @Override
    public void commit(WriterCommitMessage[] messages) {
      AppendFiles append = table.newAppend();

      int numFiles = 0;
      for (DataFile file : files(messages)) {
        numFiles += 1;
        append.appendFile(file);
      }

      commitOperation(append, String.format("append with %d new data files", numFiles));
    }
  }
```

SnapshotProducer 的 commit() 也比较简单先生成 snapshot ,然后交给 TableOperations 向catalog 提交
生成 snapshot 需要写入 manifestList 等一系列操作

```
//SnapshotProducer 代码有删减
@Override
  public void commit() {
 Snapshot newSnapshot = apply();
 taskOps.commit(base, updated.withUUID());
});

public Snapshot apply() {
  // 记录当前的 snapshotId ，base 是 TableMetadata 对象
Long parentSnapshotId = base.currentSnapshot().snapshotId();
  //生成 manifests 文件
List<ManifestFile> manifests = apply(base);
  //生成 manifestListPath 用于写入文件
OutputFile manifestList = manifestListPath();
  //生成 ManifestListWriter
ManifestListWriter writer = ManifestLists.write(
          ops.current().formatVersion(), manifestList, snapshotId(), parentSnapshotId, sequenceNumber)
writer.addAll(Arrays.asList(manifestFiles));
  //生成snapshot 
return new BaseSnapshot(ops.io(),
          sequenceNumber, snapshotId(), parentSnapshotId, System.currentTimeMillis(), operation(), summary(base),
          base.currentSchemaId(), manifestList.location());

}
```

还剩下最后一步，生成 ManifestFile 同时进行 merge, 生成ManifestFile的流程要复杂很多

1.  先从当前的 snapshot 中过滤出还可以使用的 ManifestFile，由于是 append 模式，所以之前的 ManifestFile 都可以继续使用
2.  通过 prepareNewManifests 生成 新加入的 DataFile 的 ManifestFile
3.  将新生成的和 从上次过滤出来的 ManifestFile 做一个 concat
4.  对 ManifestFile 内容按分区做合并，如果需要会对 manifest 文件重写，同时减少 Manifest 文件的数量，这一块逻辑比较复杂，交给读者自行分析

```
//MergingSnapshotProducer 代码有删减
public List<ManifestFile> apply(TableMetadata base) {
List<ManifestFile> filtered = filterManager.filterManifests(base.schema(),current.dataManifests());
Iterable<ManifestFile> unmergedManifests = Iterables.filter(
        Iterables.concat(prepareNewManifests(), filtered), shouldKeep);
Iterables.addAll(manifests, mergeManager.mergeManifests(unmergedManifests));
return manifests;

}
```

prepareNewManifests 通过 ManifestWriter 将 DataFile 文件写入到磁盘

```
//MergingSnapshotProducer 代码有删减
private Iterable<ManifestFile> prepareNewManifests() {
    Iterable<ManifestFile> newManifests;
    if (newFiles.size() > 0) {
      ManifestFile newManifest = newFilesAsManifest();
      newManifests = Iterables.concat(ImmutableList.of(newManifest), appendManifests, rewrittenAppendManifests);
    } 
    return Iterables.transform(
        newManifests,manifest -> GenericManifestFile.copyOf(manifest).withSnapshotId(snapshotId()).build());
  }

private ManifestFile newFilesAsManifest() {
    ManifestWriter<DataFile> writer = newManifestWriter(writeSpec());
    try {
          writer.addAll(newFiles); // 将 DataFile 对象写入到文件，生成 ManifestFile
        } finally {
          writer.close();
    }
 this.cachedNewManifest = writer.toManifestFile();
    }
    return cachedNewManifest;
  }
```

最后我们简单看一下 生成的 manifest.avro 文件，我们通过 avro tool 将内容转换成了 JSON

![[Picture/6983ee456b4f947fc08312aaa909eb4f_MD5.jpg]]

我们再数据写入流程中提到, 会根据不同的语句 生成不同的 `BatchWrite` 子类，对应到 manifest 管理最终也会生成不同的 PendingUpdate 子类。

整个流程和分析框架都是一样的，读者可自行打断点进行分析

```
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

至此 spark 的数据写入流程已经分析完了，如果还有时间会单独再写用spark 删除和更新数据的流程

下一篇我们将分析 spark 读数据流程。gg