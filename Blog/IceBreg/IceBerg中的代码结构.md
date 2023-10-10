# Action代码结构


**Action类的树形层次图**
```
Action (org.apache.iceberg.actions)
    BaseAction (org.apache.iceberg.actions)
        BaseSnapshotUpdateAction (org.apache.iceberg.actions)
            BaseDropIndexDataAction (org.apache.iceberg.actions)
            BaseOptimizeTableAction (org.apache.iceberg.actions)
                OptimizeTableAction (org.apache.iceberg.actions)
                    Spark3OptimizeTableAction (org.apache.iceberg.actions)
            BaseRewriteDataFilesAction (org.apache.iceberg.actions)
                RewriteDataFilesAction (org.apache.iceberg.flink.actions)
            BaseWriteAggIndicesAction (org.apache.iceberg.actions)
                Spark3WriteAggIndicesAction (org.apache.iceberg.actions)
            BaseWriteIndicesAction (org.apache.iceberg.actions)
                Spark3WriteIndicesAction (org.apache.iceberg.actions)
            DropAggIndexFileAction (org.apache.iceberg.actions)
        BaseWriteColumnDictionaryAction (org.apache.iceberg.actions)
            Spark3WriteColumnDictionaryAction (org.apache.iceberg.actions)
    BaseSparkAction (org.apache.iceberg.spark.actions)
        BaseDeleteOrphanFilesSparkAction (org.apache.iceberg.spark.actions)
        BaseDeleteReachableFilesSparkAction (org.apache.iceberg.spark.actions)
        BaseExpireSnapshotsSparkAction (org.apache.iceberg.spark.actions)
        BaseSnapshotUpdateSparkAction (org.apache.iceberg.spark.actions)
            BaseRewriteDataFilesSparkAction (org.apache.iceberg.spark.actions)
                BaseRewriteDataFilesSpark3Action (org.apache.iceberg.spark.actions)  
            BaseRewriteManifestsSparkAction (org.apache.iceberg.spark.actions)
        BaseTableCreationSparkAction (org.apache.iceberg.spark.actions)
            BaseConvertTableSparkAction (org.apache.iceberg.spark.actions)
            BaseMigrateTableSparkAction (org.apache.iceberg.spark.actions)
            BaseSnapshotTableSparkAction (org.apache.iceberg.spark.actions)
    ConvertTable (org.apache.iceberg.actions)
        BaseConvertTableSparkAction (org.apache.iceberg.spark.actions)
    DeleteOrphanFiles (org.apache.iceberg.actions)
        BaseDeleteOrphanFilesSparkAction (org.apache.iceberg.spark.actions)
    DeleteReachableFiles (org.apache.iceberg.actions)
        BaseDeleteReachableFilesSparkAction (org.apache.iceberg.spark.actions)
    ExpireSnapshots (org.apache.iceberg.actions)
        BaseExpireSnapshotsSparkAction (org.apache.iceberg.spark.actions)
    MigrateTable (org.apache.iceberg.actions)
        BaseMigrateTableSparkAction (org.apache.iceberg.spark.actions)
    SnapshotTable (org.apache.iceberg.actions)
        BaseSnapshotTableSparkAction (org.apache.iceberg.spark.actions)
    SnapshotUpdate (org.apache.iceberg.actions)
        BaseSnapshotUpdateSparkAction (org.apache.iceberg.spark.actions)
            BaseRewriteDataFilesSparkAction (org.apache.iceberg.spark.actions)
                BaseRewriteDataFilesSpark3Action (org.apache.iceberg.spark.actions)
            BaseRewriteManifestsSparkAction (org.apache.iceberg.spark.actions)
        ConvertEqualityDeleteFiles (org.apache.iceberg.actions)
        OptimizeTable (org.apache.iceberg.actions)
        RewriteDataFiles (org.apache.iceberg.actions)
            BaseRewriteDataFilesSparkAction (org.apache.iceberg.spark.actions)
                BaseRewriteDataFilesSpark3Action (org.apache.iceberg.spark.actions)
        RewriteManifests (org.apache.iceberg.actions)
            BaseRewriteManifestsSparkAction (org.apache.iceberg.spark.actions)
        RewritePositionDeleteFiles (org.apache.iceberg.actions)
    SnapshotUpdateAction (org.apache.iceberg.actions)
        BaseSnapshotUpdateAction (org.apache.iceberg.actions)
            BaseDropIndexDataAction (org.apache.iceberg.actions)
            BaseOptimizeTableAction (org.apache.iceberg.actions)
                OptimizeTableAction (org.apache.iceberg.actions)
                    Spark3OptimizeTableAction (org.apache.iceberg.actions)
            BaseRewriteDataFilesAction (org.apache.iceberg.actions)
                RewriteDataFilesAction (org.apache.iceberg.flink.actions)
            BaseWriteAggIndicesAction (org.apache.iceberg.actions)
                Spark3WriteAggIndicesAction (org.apache.iceberg.actions)
            BaseWriteIndicesAction (org.apache.iceberg.actions)
                Spark3WriteIndicesAction (org.apache.iceberg.actions)
            DropAggIndexFileAction (org.apache.iceberg.actions)
```

**各类的介绍**
Action:
	BaseAction
	BaseSparkAction
	CovertTable
	DeleteOrphanFiles
	DeleteReachableFiles
	ExpireSnashots


# Procedure代码结构

**Procedure(程序）类的树形层次图**
```
Procedure (org.apache.spark.sql.connector.iceberg.catalog)
    BaseProcedure (org.apache.iceberg.spark.procedures)
        AddFilesProcedure (org.apache.iceberg.spark.procedures)
        AncestorsOfProcedure (org.apache.iceberg.spark.procedures)
        CherrypickSnapshotProcedure (org.apache.iceberg.spark.procedures)
        ConvertTableProcedure (org.apache.iceberg.spark.procedures)
        ExpireSnapshotsProcedure (org.apache.iceberg.spark.procedures)
        MigrateTableProcedure (org.apache.iceberg.spark.procedures)
        RemoveAggIndexFilesProcedure (org.apache.iceberg.spark.procedures)
        RemoveIndexFilesProcedure (org.apache.iceberg.spark.procedures)
        RemoveOrphanFilesProcedure (org.apache.iceberg.spark.procedures)
        RevertConvertedTableProcedure (org.apache.iceberg.spark.procedures)
        RewriteDataFilesProcedure (org.apache.iceberg.spark.procedures)
        RewriteManifestsProcedure (org.apache.iceberg.spark.procedures)
        RollbackToSnapshotProcedure (org.apache.iceberg.spark.procedures)
        RollbackToTimestampProcedure (org.apache.iceberg.spark.procedures)
        SetCurrentSnapshotProcedure (org.apache.iceberg.spark.procedures)
        SnapshotTableProcedure (org.apache.iceberg.spark.procedures)
        WriteAggIndexFilesProcedure (org.apache.iceberg.spark.procedures)
        WriteIndexFilesProcedure (org.apache.iceberg.spark.procedures)

```

**各类的介绍**
Procedure:
	BaseProcedure:
		



流程：
1. IcebergSparkSessionExtensions 在计划阶段进行插入相应的Plan拓展；
2. ExtendedDataSourceV2Strategy 利用逻辑执行计划进行匹配，若语法树节点匹配到Call函数的操作，则调用Procedure中的call方法；
3. Procedure调用call方法去执行具体逻辑；
4. Procedure中包含对象Action对象，每个Action在call方法中执行execute()方法，执行具体的逻辑；

## Procedure 与 Action 的对应关系

| NO  | Procedure                    | Action                           |
| --- | ---------------------------- | -------------------------------- |
| 1   | ConvertTableProcedure        | BaseConvertTableSparkAction      |
| 2   | ExpireSnapshotsProcedure     | BaseExpireSnapshotsSparkAction   |
| 3   | MigrateTableProcedure        | BaseMigrateTableSparkAction      |
| 4   | RemoveAggIndexFilesProcedure | DropAggIndexFileAction           |
| 5   | RemoveIndexFilesProcedure    | BaseDropIndexDataAction          |
| 6   | RemoveOrphanFilesProcedure   | BaseDeleteOrphanFilesSparkAction |
| 7   | RewriteDataFilesProcedure    | BaseRewriteDataFilesSpark3Action |
| 8   | RewriteManifestsProcedure    | BaseRewriteManifestsSparkAction  |
| 9   | SnapshotTableProcedure       | BaseSnapshotTableSparkAction     |
| 10  | WriteAggIndexFilesProcedure  | BaseWriteAggIndicesAction        |
| 11  | WriteIndexFilesProcedure     | BaseWriteIndicesAction           |                                  |


**其他的execute()方法对应的Action**
1. Spark3WriteAggIndicesAction的mayUpdateColumnDictionary方法中调用   -->   BaseWriteColumnDictionaryAction
2. 



# TableScan代码结构

**TableScan类的树形层次图**
```
TableScan（org.apache.iceberg）
    BaseTableScan (org.apache.iceberg)
        BaseMetadataTableScan (org.apache.iceberg)
            BaseAllMetadataTableScan (org.apache.iceberg)
                AllDataFilesTableScan in AllDataFilesTable (org.apache.iceberg)
                AllManifestsTableScan in AllManifestsTable (org.apache.iceberg)
                Scan in AllEntriesTable (org.apache.iceberg)
            EntriesTableScan in ManifestEntriesTable (org.apache.iceberg)
            FilesTableScan in DataFilesTable (org.apache.iceberg)
            StaticTableScan (org.apache.iceberg)
                HistoryScan in HistoryTable (org.apache.iceberg)
                ManifestsTableScan in ManifestsTable (org.apache.iceberg)
                PartitionsScan in PartitionsTable (org.apache.iceberg)
                SnapshotsTableScan in SnapshotsTable (org.apache.iceberg)
        DataTableScan (org.apache.iceberg)
            IncrementalDataTableScan (org.apache.iceberg)

```



# Table代码结构

**Table类的树形层次图**
```
Table (org.apache.iceberg)
    BaseMetadataTable (org.apache.iceberg)
        AllDataFilesTable (org.apache.iceberg)
        AllEntriesTable (org.apache.iceberg)
        AllManifestsTable (org.apache.iceberg)
        DataFilesTable (org.apache.iceberg)
        HistoryTable (org.apache.iceberg)
        ManifestEntriesTable (org.apache.iceberg)
        ManifestsTable (org.apache.iceberg)
        PartitionsTable (org.apache.iceberg)
        SnapshotsTable (org.apache.iceberg)
    BaseTable (org.apache.iceberg)
        Anonymous in writeAndGetAppender() in TestSparkMergingMetrics (org.apache.iceberg.spark.source)
        TestTable in TestTables (org.apache.iceberg)
        TestTable in TestTables (org.apache.iceberg.spark.source)
    SerializableTable (org.apache.iceberg)
        SerializableMetadataTable in SerializableTable (org.apache.iceberg)
    TransactionTable in BaseTransaction (org.apache.iceberg)
```


IcebergSparkSessionExtensions： 用与在在SparkSession中插入优化规则
	1：编译阶段：
	2：解析阶段：
	3：优化阶段：
	4：计划阶段：


**rewriteFiles函数的调用过程**
BaseRewriteDataFilesSparkAction#execute
BaseRewriteDataFilesSparkAction#doExecuteWithPartialProgress
RewriteDataFilesCommitManager#start
RewriteDataFilesCommitManager#commitOrClean
RewriteDataFilesCommitManager#commitFileGroups
BaseRewriteFiles


BaseRewriteDataFilesSparkAction#execute
BaseRewriteDataFilesSparkAction#doExecute
RewriteDataFilesCommitManager#commitOrClean
RewriteDataFilesCommitManager#commitFileGroups
BaseRewriteFiles


**BaseRewriteFiles方法代码解读**
```
public RewriteFiles rewriteFiles(
								Set<DataFile> dataFilesToReplace, 
								Set<DeleteFile> deleteFilesToReplace,  
                                Set<DataFile> dataFilesToAdd, 
	                            Set<DeleteFile> deleteFilesToAdd) { 
	// 验证输入输出，主要对输出的参数进行校验，删除为文件必须存在，但是新增的文件可以为null
  verifyInputAndOutputFiles(dataFilesToReplace, deleteFilesToReplace, dataFilesToAdd, deleteFilesToAdd); 
  // 将要删除的dataFile文件全部存放在一个set中 
  replacedDataFiles.addAll(dataFilesToReplace);  

  // 删除文件
  for (DataFile dataFile : dataFilesToReplace) {  
    delete(dataFile);  
  }  
  for (DeleteFile deleteFile : deleteFilesToReplace) {  
    delete(deleteFile);  
  } 
   
  // 增加文件
  for (DataFile dataFile : dataFilesToAdd) {  
    add(dataFile);  
  }  
  for (DeleteFile deleteFile : deleteFilesToAdd) {  
    add(deleteFile);  
  }  
  
  return this;  
}
```


# TableOperations代码结构
负责表的元数据操作。表的元数据操作需要与Catalog进行交互操作，每一个Catalog有相应的TableOperation具体实现。

**TableOperations类的树形层次图**
```
TableOperations (org.apache.iceberg)
    Anonymous in temp() in BaseMetastoreTableOperations (org.apache.iceberg)
    Anonymous in temp() in HadoopTableOperations (org.apache.iceberg.hadoop)
    BaseMetastoreTableOperations (org.apache.iceberg)
        DynamoDbTableOperations (org.apache.iceberg.aws.dynamodb)
        GlueTableOperations (org.apache.iceberg.aws.glue)
        HiveTableOperations (org.apache.iceberg.hive)
        JdbcTableOperations (org.apache.iceberg.jdbc)
        NessieTableOperations (org.apache.iceberg.nessie)
    HadoopTableOperations (org.apache.iceberg.hadoop)
    LocalTableOperations (org.apache.iceberg)
    StaticTableOperations (org.apache.iceberg)
    TestTableOperations in TestTables (org.apache.iceberg)
        Anonymous in opsWithCommitSucceedButStateUnknown() in TestTables (org.apache.iceberg)
    TestTableOperations in TestTables (org.apache.iceberg.spark.source)
    TransactionTableOperations in BaseTransaction (org.apache.iceberg)

```
核心方法： 
1. current() 通过 Catalog 加载出当前表的 metadata 数据
2. commit() 数据写入完成后提交当前表，也就是生成一个 snapshot
3. io() 表示当前表的底层存储介质，比如 HDFS, AWS


# Catalog代码结构


**Catalog类的树形代码结构**
```
Catalog (org.apache.iceberg.catalog)
    BaseMetastoreCatalog (org.apache.iceberg)
        DynamoDbCatalog (org.apache.iceberg.aws.dynamodb)
        GlueCatalog (org.apache.iceberg.aws.glue)
        HadoopCatalog (org.apache.iceberg.hadoop)
            CustomHadoopCatalog in TestCatalogs (org.apache.iceberg.mr)
            CustomHadoopCatalog in TestFlinkCatalogFactory (org.apache.iceberg.flink)
        HiveCatalog (org.apache.iceberg.hive)
        JdbcCatalog (org.apache.iceberg.jdbc)
        NessieCatalog (org.apache.iceberg.nessie)
        TestCatalog in TestCatalogUtil (org.apache.iceberg)
        TestCatalogBadConstructor in TestCatalogUtil (org.apache.iceberg)
        TestCatalogConfigurable in TestCatalogUtil (org.apache.iceberg)
        TestCatalogErrorConstructor (org.apache.iceberg)
    CachingCatalog (org.apache.iceberg)
        TestableCachingCatalog (org.apache.iceberg)

```


# File代码结构
ContentFile类的树型代码结构
```
ContentFile(org.apache.iceberg)
    BaseFile (org.apache.iceberg)
        GenericDataFile (org.apache.iceberg)
        GenericDeleteFile (org.apache.iceberg)
    DataFile (org.apache.iceberg)
        GenericDataFile (org.apache.iceberg)
        IndexedDataFile in V1Metadata (org.apache.iceberg)
        SparkDataFile (org.apache.iceberg.spark)
        TestDataFile in TestHelpers (org.apache.iceberg)
    DeleteFile (org.apache.iceberg)
        GenericDeleteFile (org.apache.iceberg)
    IndexedDataFile in V2Metadata (org.apache.iceberg)
```

# StructLike代码结构
StructLike类的树型代码结构
```
SparkStructLike (org.apache.iceberg)
    AggIndexData (org.apache.iceberg)
    BaseFile (org.apache.iceberg)
        GenericDataFile (org.apache.iceberg)
        GenericDeleteFile (org.apache.iceberg)
    GenericManifestEntry (org.apache.iceberg)
    GenericManifestFile (org.apache.iceberg)
    GenericPartitionFieldSummary (org.apache.iceberg)
    GenericRecord (org.apache.iceberg.data)
    IndexData (org.apache.iceberg)
    IndexedStructLike (org.apache.iceberg)
    InternalRecordWrapper (org.apache.iceberg.data)
    InternalRowWrapper (org.apache.iceberg.spark.source)
    LocalDateTimeToLongMicros in ArrowReaderTest (org.apache.iceberg.arrow.vectorized)
    PartitionData (org.apache.iceberg)
        Anonymous in EMPTY_PARTITION_DATA in BaseFile (org.apache.iceberg)
    PartitionKey (org.apache.iceberg)
    PositionDelete (org.apache.iceberg.deletes)
    Record (org.apache.iceberg.data)
        GenericRecord (org.apache.iceberg.data)
    Row in StaticDataTask (org.apache.iceberg)
    Row in TestHelpers (org.apache.iceberg)
    RowDataWrapper (org.apache.iceberg.flink)
    SparkStructLike (org.apache.iceberg.spark)
    StructCopy (org.apache.iceberg.io)
    StructProjection (org.apache.iceberg.util)
```


# BatchWrite代码结构
**BatchWrite类的树型代码结构**
```
BatchWrite (org.apache.spark.sql.connector.write)
    AggIndexBatchWrite (org.apache.iceberg.spark.aggindex)
    BaseBatchWrite in SparkWrite (org.apache.iceberg.spark.source)
        BatchAppend in SparkWrite (org.apache.iceberg.spark.source)
        CopyOnWriteMergeWrite in SparkWrite (org.apache.iceberg.spark.source)
        DynamicOverwrite in SparkWrite (org.apache.iceberg.spark.source)
            SerializedDynamicOverwrite in SparkWrite (org.apache.iceberg.spark.source)
        OverwriteByFilter in SparkWrite (org.apache.iceberg.spark.source)
        RewriteFiles in SparkWrite (org.apache.iceberg.spark.source)
    FileBatchWrite (org.apache.spark.sql.execution.datasources.v2)
    MicroBatchWrite (org.apache.spark.sql.execution.streaming.sources)
    NoopBatchWrite$ (org.apache.spark.sql.execution.datasources.noop)

```


# 疑问：
IceBerg中如何解决并发commit的问题

