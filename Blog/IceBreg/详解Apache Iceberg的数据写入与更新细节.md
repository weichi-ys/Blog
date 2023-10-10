Apache Iceberg有着复杂的树形元数据层，这个既是其优点又是其缺点。

随着AI大模型的发展，数据中心也在向AI靠拢—为AI的发展提供助力。传统的Hadoop系的数据湖，对数据基本上只提供了Append的能力，对于Upsert与ACID能力是不支持的。然而在算法模型的迭代中，数据特征也一直在迭代更新，这也要求数据湖能力提供快速可更迭且向前兼容的数据特征存储。

我们就借着这个背景，来详细解释下Iceberg是如何实现更新的。

在介绍更新之前，我们先来简单介绍下Iceberg的写入流程。

首先，我们创建一个Order表，其分区为HOUR(order\_ts)，表示该表按照`order_ts`的小时粒度进行分区：

```
CREATE TABLE orders (
   order_id BIGINT,
   customer_id BIGINT,
   order_amount DECIMAL(10, 2),
   order_ts TIMESTAMP
) 
USING iceberg
PARTITIONED BY (HOUR(order_ts))
```

然后我们执行数据的insert操作。请注意我们操作表时不必添加显式列来使用 Iceberg 表进行分区，iceberg支持隐式分区。

```
## Spark SQL
INSERT INTO orders VALUES (
   123,
   456,
   36.17,
   '2023-03-07 08:10:23'
)
```

### **1\. 引擎的解析与执行**

Iceberg定位只是一个表格式，SQL的执行都是依赖于引擎。Insert语句首先会被引擎进行解析构图，生成执行计划。在这个过程中，引擎在解析到orders表时需要关联真正的物理表，这时就需要拿到表的信息。

### **2\. 检查Catalog**

Iceberg在设计之初就是独立于任何引擎，它设计了一套自己的Catalog与schema。好处就是方便了后续不依赖任何引擎的独立更新迭代—没有了羁绊。缺点就是会有点复杂，有点性能损耗，不过基本可以忽略。

Iceberg的入口就是其Catalog，在写入时，执行引擎向Catalog发出请求以确定当前元数据文件的位置，然后读取它。

假如我们使用的是 Hadoop Catalog，这时引擎将读取该`/orders/metadata/version-hint.txt`文件并发现该文件的内容是 1。利用Catalog实现中的逻辑，引擎知道当前元数据文件位置位于`/orders/metadata/v1.metadata.json`

虽然我们引擎的动机是想插入新的数据文件，但它仍然需要先与Catalog进行交互，这样做的原因有两个：

-   **引擎需要了解表的模式（v1, v2等架构）；**
-   **了解表的分区方案，以便在写入时相应地组织数据；**

### **3\. 写入数据文件**

首先，引擎根据表的分区定义方案方案将数据写入 Parquet 数据文件。此外，如果为表定义了排序顺序，则数据将在写入数据文件之前进行排序。下面就是写入后的数据文件：

```
s3://datalake/db1/orders/data/order_ts_hour=2023-03-07-08/0_0_0.parquet
```

### 4\. 创建**元数据文件**

在写入数据文件后，引擎会创建一个清单文件。

该清单文件提供了有关引擎创建的实际数据文件的路径的信息和统计信息。统计信息包括有：列的上下界、空值计数等，这非常有利于查询引擎修剪文件并提供最佳性能。

需要注意的是，引擎在处理要写入的数据时统计这些信息，这是一个相对轻量级的操作。最终，清单文件将作为 .avro 文件写入存储系统中。

```
s3://datalake/db1/orders/metadata/62acb3d7-e992-4cbc-8e41-58809fcacb3e.avro
```

下面这是清单文件的部分摘要内容：

```
{
      "data_file" : {
      "file_path" : 
“s3://datalake/db1/orders/data/order_ts_hour=2023-03-07-08/0_0_0.parquet”,

      "file_format" : "PARQUET",
      "block_size_in_bytes" : 67108864,
      "null_value_counts" : [],
      "lower_bounds" : {
         "array": [{
         "key": 1,
         "value": 123
                 }],
      }
     "upper_bounds" : {
         "array": [{
         "key": 1,
         "value": 123
                 }],
      },
   }
}
```

接下来，引擎创建一个清单列表来跟踪清单文件。

如果现有清单文件与当前快照有关联的，那么这些文件也将被添加到这个新的清单列表中。

然后引擎会将此文件写入数据湖中，其中包含清单文件的路径、添加或删除的数据文件/行数以及有关分区的统计信息（例如分区列的下限和上限）等信息。

同样，引擎已经拥有所有这些信息，因此获取这些统计数据是一项轻量级操作。此信息有助于读取查询排除任何不需要的清单文件，从而促进更快的查询。

```
s3://datalake/db1/orders/metadata/snap-8333017788700497002-1-4010cc03-5585-458c-9fdc-188de318c3e6.avro
```

这是清单列表内容的片段:

```
{
 "manifest_path": 
"s3://datalake/db1/orders/metadata/62acb3d7-e992-4cbc-8e41-58809fcacb3e.avro",
 "manifest_length": 6152,
 "added_snapshot_id": 8333017788700497002,
 "added_data_files_count": 1,
 "added_rows_count": 1,
 "deleted_rows_count": 0,
 "partitions": {
        "array": [ {
            "contains_null": false,
            "lower_bound": {
                "bytes": "¹Ô\\\\u0006\\\\u0000"
            },
            "upper_bound": {
                "bytes": "¹Ô\\\\u0006\\\\u0000"
            }
        } ]
    }
}
```

最后，引擎通过现有元数据文件`v1.metadata.json`跟踪到先前的快照 s0。然后将s0和当前创建的新快照s1作为一部分创建新的元数据文件 `v2.metadata.json` ，即`v2.metadata.json`。同时跟踪s0，s1。

这个新的元数据文件包含有关引擎创建的清单列表的信息，其中包含清单列表文件路径、快照 ID、操作摘要等详细信息。此外，引擎还将引用当前的清单列表（或快照）。

```
s3://datalake/db1/orders/metadata/v2.metadata.json
```

这个新元数据文件的内容如下所示:

```
"current-snapshot-id" : 8333017788700497002,
 "refs" : {
 "main" :  {
 "snapshot-id" : 8333017788700497002,
      "type" : "branch"
      }
},
"snapshots" : [ {
    "snapshot-id" : 8333017788700497002,
    "summary" : {
      "operation" : "append",
      "added-data-files" : "1",
      "added-records" : "1",
},
    "manifest-list" : “s3://datalake/db1/orders/metadata/snap-8333017788700497002-1-4010cc03-5585-458c-9fdc-188de318c3e6.avro”,
  } ],
```

### **5\. 更新Catalog并commit改变**

最后，引擎再次访问Catalog，以确保运行此 INSERT 操作时没有提交其他快照。

通过进行此验证，Iceberg 保证在多个写入者同时写入数据的场景中不会干扰操作。Iceberg 确保第一个写入数据的人将首先提交，任何冲突的写入操作将返回到之前的步骤并重新尝试，直到写入成功或失败。

最后，引擎自动更新Catalog以引用新的元数据`v2.metadata.json`，该元数据现在成为当前的元数据文件。

整体的运行流程如下图所示：

![[Picture/40ab529711b1bb49f82cc615c599570e_MD5.jpg]]

## 2\. Iceberg 数据更新

如果从本质上讲数据湖并没有提供update的能力，它实际上就是通过Join来实现的数据更新。

直接将增量或全量更新的能力写到Spark或其他引擎也是可以的。

如果直接在Spark中采用原始的Join来实现update可能效率上有点差，这就需要对其进行定制性的优化，例如增加些中间文件或统计文件，这会使得Spark更加复杂，更难上手。

总的来说，数据湖对Update能力的支持与优化就是对Join的支持优化。那么Join怎么就更快了呢？比如排序，分桶，文件大小等等。这也使得数据湖要想update运算快，就需要很多配置的针对性调整。

下面我们说回Iceberg，来看下Iceberg是如何实现UPSERT/MERGE INTO的。

如下SQL语句，当想要更新现有行（如果表中存在特定值）时，通常会运行此类查询，如果不存在，则只需插入新行。

```
## Spark SQL
MERGE INTO orders o
USING (SELECT * FROM orders_staging) s
ON o.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET order_amount = s.order_amount
WHEN NOT MATCHED THEN INSERT *;
```

假设有一个阶段表 `orders_staging`它由两条记录组成。 一条记录对现有`order_id`( `order_id=123`) 进行了更新，另一条记录是全新的订单。

我们希望使用每个订单的最新详细信息更新订单表，因此如果当前表中的`order_id`在目标表中已存在，我们将更新`orders`中的order\_amount，如果没有，我们将只插入新数据。

### **1\. 引擎的解析与执行**

同样的我们需要向引擎发送SQL，引擎会进行解析查询计划。由于涉及两个表（`orders_staging`和orders），引擎需要两个表的数据来开始查询计划。

### **2\. 检查Catalog**

与 INSERT 操作类似，查询引擎首先向Catalog发出请求以确定当前元数据文件位置，然后读取它。由于本测试使用的是 Hadoop Catalog，因此引擎将读取`/orders/metadata/version-hint.txt file`并检索文件的内容，即整数 2。引擎获知当前元数据文件位置是`/orders/metadata/v2.metadata.json`。

因此，引擎将读取该文件。然后它将查看表的当前模式与分区信息，以便写入操作可以遵循它。

### **3\. 写入数据文件**

首先，查询引擎将从`orders_staging`和orders中读取数据并将其加载到内存中，以确定匹配的数据记录。

当引擎确定匹配时，内存中跟踪的数据内容将基于 Iceberg 表属性定义的两种策略(写入时复制COW或读取时合并MOR)进行写出。

写时复制策略，每当更新 Iceberg 表时，任何具有相关记录的关联数据文件都将被重写为新数据文件。然而，使用读时合并，数据文件不会被重写；相反，将生成新的删除文件来跟踪更改。

这里我们将先按写时复制策略走完整个流程，然后再详细解释下读取时合并策略实现。

因此在`0_0_0.parquet`中包含`order_id = 123`的数据记录将被读入内存。然后，数据记录中的`order_amount`该字段将被数据的内存副本中表`order_staging`的`order_id`新`order_amount`字段进行更新。最后，这些修改后的详细信息将写入新的parquet数据文件中。

```
s3://datalake/db1/orders/data/order_ts_hour=2023-03-07-08/0_0_1.parquet
```

现在，表中不符合条件的记录`order_staging`将被视为常规 INSERT，并将作为新数据文件写入不同的分区。这里被写入新的分区order\_ts\_hour=2023-01-27-10。

```
s3://datalake/db1/orders/data/order_ts_hour=2023-01-27-10/0_0_0.parquet
```

### **4\. 生成元数据文件**

写入数据文件后，引擎会创建一个新的清单文件，其中包含对这两个数据文件的文件路径的引用。此外，清单文件中还包含有关这些数据文件的各种统计信息。

```
s3://datalake/db1/orders/metadata/faf71ac0-3aee-4910-9080-c2e688148066.avro
```

这是清单文件的一个片段:

```
{
              "data_file" : {
              "file_path" : 
        "s3://datalake/db1/orders/data/order_ts_hour=2023-01-27-10/0_0_0.parquet",
              "file_format" : "PARQUET",
              "block_size_in_bytes" : 67108864,
              "null_value_counts" : [],
              "lower_bounds" : {
                    "array": [{
                    "key": 1,
                    "value": 125
                         }],
               }
               "upper_bounds" : {
                     "array": [{
                     "key": 1,
                     "value": 125
                          }],
               }
            },
               "data_file" : {
               "file_path" : 
         "s3://datalake/db1/orders/data/order_ts_hour=2023-03-07-08/0_0_1.parquet",
               "file_format" : "PARQUET",
               "block_size_in_bytes" : 67108864,
               "null_value_counts" : [],
               "lower_bounds" : {
                     "array": [{
                     "key": 1,
                     "value": 123
                          }],
                }
                "upper_bounds" : {
                      "array": [{
                      "key": 2,
                      "value": 200
                           }],
                      }
                }
        }
```

然后，引擎生成一个新的清单列表，该列表指向上一步中创建的清单文件。

```
s3://datalake/db1/orders/metadata/snap-5139476312242609518-1-e22ff753-2738-4d7d-a810-d65dcc1abe63.avro
```

和插入类似，清单列表还包括分区统计信息、添加和删除文件的数量等信息。

之后，引擎将通过当前元数据文件`v2.metadata.json`和跟踪到其快照`s0`与`s1`，以及新快照 s2 ，并为他们创建新元数据文件`v3.metadata.json`。

```
s3://datalake/db1/orders/metadata/v3.metadata.json
```

### **5\. 更新Catalog并commit改变**

最后，引擎此时运行检查以确保不存在写入冲突，然后使用最新元数据文件的值（即`v3.metadata.json`

整体的Merge Into运行流程如下图所示：

![[Picture/e18e725dfd294b356b518fd03cf4de52_MD5.jpg]]

## 3\. Iceberg 读时合并更新原理

顾名思义，读时合并就是在写入时只写入要更新的数据，在读取时通过Join将文件进行合并读取（也可在写入后的后台线程进行文件合并），拿到更新后的数据。

### **1\. 两个重要的概念**

1\. **delete file**：（删除文件）描述了在读取数据时那些需要被删除的行的数据集， 它可以使用基于位置的数据集(position-based delete file)来描述，也可以使用基于值数据集(value-based delete file)来描述，删除文件的格式和原数据文件的格式一致，可以同样进行信息统计实现过滤谓词下推。delete file 和data file 文件类型靠 **content** 字段区分。

2\. **sequence number**: （序列号）描述Iceberg文件的顺序数，序列号越小，生成该文件的时间越早。它决定了删除文件是否应该和对应的数据文件进行合并，当删除文件的序列号大于数据文件的序列号时，需要进行数据合并。

### **2\. Delete File 实现行级删除**

要了解Iceberg的读取合并策略，还需要再详细解释下Iceberg的数据层。

Iceberg的数据层包括：**Data File, Delete File 和Puffin Files。**

**Delete File又包括Positional Delete Files和Equality Delete Files。**

1.  **Positional Delete Files**

位置删除文件，它通过指定包含该行的特定文件的文件路径以及该文件中的行号来标明哪些行已被逻辑删除。如下图所示：

![[Picture/aa3e62a644af7e0969f31b60afac6e2b_MD5.jpg]]

左边为原始的数据文件，在执行Delete删除语句后，在右边形成一个位置删除文件。如图中SQL语句所示，我们要从订单表中删除order\_id=’1234‘数据。假设文件中的数据按`order_id`升序排序，该数据位于文件 #2 中，并且是行 #234。因此，我们生成了位置删除文件的一行内容为:

注意：这里只是形象的表示，具体的路径path应该是具体的地址。

1.  **Equality Delete Files**

相等删除文件，就是通过记录要删除的条件，在读取时通过匹配每一行来表示删除。方法是通过一个或多个字段的值来识别行，也可以通过此方法删除多行。如下图所示：

![[Picture/1662727e30d2d3c022c04626dd00a4b7_MD5.jpg]]

同样是从订单表中删除order\_id=’1234‘数据，这里生成的相等删除文件中，只需要记录”order\_id = 1234”即可，在读取或合并时会遵守规则进行过滤或删除对应的数据。

### **3\. 合并读取细节流程**

![[Picture/30d1dfc40463310bcfbf3f462436088d_MD5.jpg]]

1\. 选择data file文件中序列号最小（假设序列号为Seq0）的文件。 任何小于或等于 Seq0序列号 的delete file都可以丢弃。  
2\. 对于序列号等于 Seq0 的每个data file数据文件，过滤掉position delete file中提到的记录的所有行（不考虑序列号），并合并结果。  
3\. 对于每个具有数据或删除文件的后续序列号：  
a. 将之前的结果与每个具有当前序列号的 equality delete file进行anti-join（对删除文件中的所有列使用等式连接）。  
b. 使用position delete file（不考虑序列号）过滤所有具有当前序列号的数据文件。  
c. 合并这两个步骤的结果。  

后面我们再单独以Flink upsert来举例吧。

## 4\. 总结

整体上我们梳理了Iceberg的insert与merge into的执行流程：

1.  **引擎的解析与执行，解析向iceberg请求获取iceberg表信息；**
2.  **检查Catalog，获取表模式，分区信息，以及当前的metadata的位置；**
3.  **写入数据文件，对于insert直接写入即可，merge into需要考虑不同模式；**
4.  **生成元数据文件，在写入过程中收集统计信息，生成统计信息文件；**
5.  **更新Catalog并commit改变，确保不存在写入冲突**。

而对于iceberg读时合并的策略，我们这里只对实现原理做了简要总结：

其实现依赖于**Delete File和sequence number，**简要总结如下**。**

**1\. 选择序列号最小的数据文件，丢弃所有序列号小于或等于该序列号的删除文件。  
2\. 对于序列号等于最小序列号的数据文件，根据位置删除文件进行过滤，删除需要逻辑删除的行。  
3\. 针对每个后续序列号的数据文件：  
a. 与相等删除文件进行反连接，去除符合删除条件的行。  
b. 使用位置删除文件过滤需要逻辑删除的行。  
c. 合并删除操作后的结果。**