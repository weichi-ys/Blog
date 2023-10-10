## 原生 Spark 对 DDL 的支持

在 Spark 3.0 中，语法文件已经支持了 DML(删除，修改) 语句，但并没有做相应的实现

```
//SqlBase.g4
dmlStatementNoWith
    : insertInto queryTerm queryOrganization                                       #singleInsertQuery
    | fromClause multiInsertQueryBody+                                             #multiInsertQuery
    | DELETE FROM multipartIdentifier tableAlias whereClause?                      #deleteFromTable
    | UPDATE multipartIdentifier tableAlias setClause whereClause?                 #updateTable
    | MERGE INTO target=multipartIdentifier targetAlias=tableAlias
        USING (source=multipartIdentifier |
          '(' sourceQuery=query')') sourceAlias=tableAlias
        ON mergeCondition=booleanExpression
        matchedClause*
        notMatchedClause*                                                          #mergeIntoTable
    ;
```

在生成 逻辑计划 时明确写了暂不支持

```
//SparkStrategies 文件
object BasicOperators extends Strategy {
    def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
case _: UpdateTable =>
        throw new UnsupportedOperationException(s"UPDATE TABLE is not supported temporarily.")
      case _: MergeIntoTable =>
        throw new UnsupportedOperationException(s"MERGE INTO TABLE is not supported temporarily.")
}
}
```

Delete 语句虽然提供了实现，但只支持 `v2 tables`

```
case class DeleteFromTableExec(
    table: SupportsDelete,
    condition: Array[Filter],
    refreshCache: () => Unit) extends V2CommandExec {

  override protected def run(): Seq[InternalRow] = {
    table.deleteWhere(condition) // 调用table 的 deleteWhere 接口
    refreshCache()
    Seq.empty
  }

  override def output: Seq[Attribute] = Nil
}
```

Spark `datasources v2` ，定义了数据读取和插入的 Api ，很方便扩展相应的操作，Spark `datasources` V2 API 后面单独开篇博客来讲解

虽然无法通过原生的 SQL 执行，可以采用自己写 SQL 的方式来实现，其实可以用等价的三条SQL语句来实现，

1.  对过滤条件取反，把数据查询出来，插入到临时表中
2.  用新插入的表替换原来的表

```
delete from local.iceberg_db.iceberg_demo where id = 1

// 等价实现
insert into local.iceberg_db.iceberg_demo_tmp  select * from local.iceberg_db.iceberg_demo 
where id <> 1
ALTER TABLE local.iceberg_db.iceberg_demo RENAME TO local.iceberg_db.iceberg_demo_delete
ALTER TABLE local.iceberg_db.iceberg_demo_tmp RENAME TO local.iceberg_db.iceberg_demo
```

可以看到自己手动写 SQL 实现，操作比较麻烦，性能也比较差，我们修改的内容可能只涉及到了一个文件，但却重写了所有文件。

![[Picture/ed19b5732e6c0f623c2ae45037a4c21b_MD5.jpg]]

结合上图我们来看一下，Iceberg 的实现思路。图中省略了Manifest 等元数据文件，S1,S2表示快照。假如我们某个表下有 3 个数据文件

1.  Iceberg 根据过滤条件和数据文件的统计信息，过滤出只包含过滤条件的数据文件，假如只包含数据文件f1
2.  在过滤出的数据文件 f1 上，对过滤条件取反进行查询，将查询结果写入新的文件
3.  将步骤一中过滤出的数据文件标记为删除，将步骤二中新写入的文件和之前不包含过滤条件的数据文件 merge 后作为新的数据文件

可以看出对于删除操作是典型的 CopyOnWrite 实现。

## Iceberg 对 SparkSQL 的扩展

在 Iceberg 的 `IcebergSparkSessionExtensions` 扩展中注入了一个规则，将逻辑计划 DeleteFromTable 转换成

Iceberg 的 RewriteDelete

```
class IcebergSparkSessionExtensions extends (SparkSessionExtensions => Unit) {

  override def apply(extensions: SparkSessionExtensions): Unit = {
    extensions.injectOptimizerRule { spark => RewriteDelete(spark) }
    extensions.injectOptimizerRule { spark => RewriteUpdate(spark) }
    extensions.injectOptimizerRule { spark => RewriteMergeInto(spark) }
}
}
```

在 RewriteDelete 我们可以看到相应的实现逻辑

1.  通过 SparkTable 的 newMergeBuilder，创建出`SparkMergeBuilder`
2.  通过 delete语句中的 condition 条件创建 scanPlan，扫描出受影响的数据文件
3.  对过滤条件取反，在受影响的数据文件上进行查询
4.  通过 `SparkWriteBuilder` 的 buildForBatch 构造出 `CopyOnWriteMergeWrite`

```
case class RewriteDelete(spark: SparkSession) extends Rule[LogicalPlan] with RewriteRowLevelOperationHelper {

  override def apply(plan: LogicalPlan): LogicalPlan = plan transform {

    case DeleteFromTable(r: DataSourceV2Relation, Some(cond)) if isIcebergRelation(r) =>
      //1. 创建出SparkMergeBuilder
      val mergeBuilder = r.table.asMergeable.newMergeBuilder("delete", writeInfo)

//2. 通说condition，扫描出受影响的数据文件
      val matchingRowsPlanBuilder = scanRelation => Filter(cond, scanRelation)
      val scanPlan = buildDynamicFilterScanPlan(spark, r, r.output, mergeBuilder, cond, matchingRowsPlanBuilder)

//3. 对过滤条件取反
      val remainingRowFilter = Not(EqualNullSafe(cond, Literal(true, BooleanType)))
      val remainingRowsPlan = Filter(remainingRowFilter, scanPlan)

//4. 通过 SparkWriteBuilder 的 buildForBatch 构造出 CopyOnWriteMergeWrite
      val mergeWrite = mergeBuilder.asWriteBuilder.buildForBatch()
      val writePlan = buildWritePlan(remainingRowsPlan, r.table, r.output)
      ReplaceData(r, mergeWrite, writePlan)
  }
}
```

我们也借此机会简单看一下 Spark 的执行，再继续往下分析。

在 RewriteDelete 中返回的是 ReplaceData，最后会生成 ReplaceDataExec 物理计划，物理计划的执行是调用 V2TableWriteExec 的 writeWithV2 方法

```
case class ReplaceDataExec(
    batchWrite: BatchWrite,
    refreshCache: () => Unit,
    query: SparkPlan) extends V2TableWriteExec {

  override protected def run(): Seq[InternalRow] = {
    prepare()
    val writtenRows = writeWithV2(batchWrite)
    refreshCache()
    writtenRows
  }
}
```

在 Spark 的执行中也是如下两步

1.  执行查询，将结果生成临时的 RDD
2.  将上一步的临时结果RDD 写入到文件

```
trait V2TableWriteExec extends V2CommandExec with UnaryExecNode {
 
  protected def writeWithV2(batchWrite: BatchWrite): Seq[InternalRow] = {
//执行查询，将结果生成临时的 RDD
    val rdd: RDD[InternalRow] = {
      val tempRdd = query.execute()
      tempRdd
    }
    //将上一步的临时结果RDD 写入到文件
    val writerFactory = batchWrite.createBatchWriterFactory(
      PhysicalWriteInfoImpl(rdd.getNumPartitions))
  
      sparkContext.runJob(
        rdd,
        (context: TaskContext, iter: Iterator[InternalRow]) =>
          DataWritingSparkTask.run(writerFactory, context, iter, useCommitCoordinator),
        rdd.partitions.indices,
        (index, result: DataWritingSparkTaskResult) => {
          val commitMessage = result.writerCommitMessage
          messages(index) = commitMessage
          batchWrite.onDataWriterCommit(commitMessage)
        }
      )

      batchWrite.commit(messages)
   }
}
```

## RewriteDelete 深度解析

接下来我们对 RewriteDelete 做深度解析

在第一步 通过 SparkTable 的 newMergeBuilder，创建出`SparkMergeBuilder`，实现了 Spark 的 `MergeBuilder` 接口，近距离看一下 `MergeBuilder` 的接口定义，可知 `MergeBuilder` 是先读后写

```
public interface MergeBuilder {
//读操作
  ScanBuilder asScanBuilder();
  //写操作
  WriteBuilder asWriteBuilder();
}
```

在 RewriteDelete 的 buildDynamicFilterScanPlan 方法中构建出了 `SparkScanBuilder` 代表读操作，读操作的流程和我们在上一篇博客中的 读操作流程一致，只是进行了两级过滤查询，1）先过滤出受影响的文件 2）对原有的过滤条件取反后再进行查询。就不具体分析。

在 mergeBuilder.asWriteBuilder.buildForBatch() 操作中，实际通过 asCopyOnWriteMergeWrite 构造出`CopyOnWriteMergeWrite`。buildWritePlan 对应的执行就是将查询后的数据进行写入

```
//SparkWriteBuilder
  @Override
  public BatchWrite buildForBatch() {

    SparkWrite write = new SparkWrite(spark, table, writeConf, writeInfo, appId, writeSchema, dsSchema);
    if (overwriteByFilter) {
      return write.asOverwriteByFilter(overwriteExpr);
    } else if (overwriteDynamic) {
      return write.asDynamicOverwrite();
    } else if (overwriteFiles) { //走的这个逻辑
      return write.asCopyOnWriteMergeWrite(mergeScan, isolationLevel);
    } else {
      return write.asBatchAppend();
    }
  }
```

## CopyOnWriteMergeWrite 分析

CopyOnWriteMergeWrite 继承自 BaseBatchWrite 已经完成了对数据的写入，BaseBatchWrite 在数据写入的博客中有具体分析。我们重点关注一下 commit 方法。

1.  通过 SparkMergeScan 找出受到过滤条件影响的文件，SparkMergeScan 继承自`SparkBatchScan`，可以参考数据读取的博客
2.  根据不同的隔离级别，通过 OverwriteFiles 进行提交 。OverwriteFiles 是`SnapshotUpdate` 的子类具体参考元数据管理和数据提交的博客
3.  隔离级别留给读者自行分析

```
private class CopyOnWriteMergeWrite extends BaseBatchWrite {
    private final SparkMergeScan scan;
    private final IsolationLevel isolationLevel;

    private List<DataFile> overwrittenFiles() {
      return scan.files().stream().map(FileScanTask::file).collect(Collectors.toList());
    }

    @Override
    public void commit(WriterCommitMessage[] messages) {
      OverwriteFiles overwriteFiles = table.newOverwrite();

//找出受到过滤条件影响的文件
      List<DataFile> overwrittenFiles = overwrittenFiles();
      int numOverwrittenFiles = overwrittenFiles.size();
      for (DataFile overwrittenFile : overwrittenFiles) {
        overwriteFiles.deleteFile(overwrittenFile);
      }

      int numAddedFiles = 0;
      for (DataFile file : files(messages)) {
        numAddedFiles += 1;
        overwriteFiles.addFile(file);
      }

//根据不同的隔离级别，通过 OverwriteFiles 进行提交
      if (isolationLevel == SERIALIZABLE) {
        commitWithSerializableIsolation(overwriteFiles, numOverwrittenFiles, numAddedFiles);
      } else if (isolationLevel == SNAPSHOT) {
        commitWithSnapshotIsolation(overwriteFiles, numOverwrittenFiles, numAddedFiles);
      } 
    }
```

MERGE INTO 和 Update 语句的实现先留给读者自行分析，后面有时间的话再补上