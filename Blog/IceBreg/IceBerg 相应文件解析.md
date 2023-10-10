# manifest文件
```
{  
    "status":2,  // 0:EXISTING; 1:ADDED; 2:DELETED
    "snapshot_id":{  
        "long":5926536096317191420  
    },  
    "data_file":{  
        "file_path":"viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/data/log_date=20230816/00046-201-aad8de32-d5a9-4a47-a82e-bba9e83b1f3c-00001.orc",  
        "file_format":"ORC",  
        "partition":{  
            "log_date":{  
                "string":"20230816"  
            }  
        },  
        "record_count":12948,  
        "file_size_in_bytes":66760329,  
        "block_size_in_bytes":67108864,  
        "column_sizes":{  
            "array":[  
                {  
                    "key":1,  
                    "value":516  
                },  
                {  
                    "key":2,  
                    "value":66655221  
                },  
                {  
                    "key":3,  
                    "value":99399  
                },  
                {  
                    "key":4,  
                    "value":47  
                }  
            ]  
        },  
        "value_counts":{  
            "array":[  
                {  
                    "key":1,  
                    "value":12948  
                },  
                {  
                    "key":2,  
                    "value":12948  
                },  
                {  
                    "key":3,  
                    "value":12948  
                },  
                {  
                    "key":4,  
                    "value":12948  
                }  
            ]  
        },  
        "null_value_counts":{  
            "array":[  
                {  
                    "key":1,  
                    "value":0  
                },  
                {  
                    "key":2,  
                    "value":0  
                },  
                {  
                    "key":3,  
                    "value":0  
                },  
                {  
                    "key":4,  
                    "value":0  
                }  
            ]  
        },  
        "nan_value_counts":{  
            "array":[  
                {  
                    "key":3,  
                    "value":0  
                }  
            ]  
        },  
        "lower_bounds":{  
            "array":[  
                {  
                    "key":1,  
                    "value":"cóÿÿÿÿÿÿ"  
                },  
                {  
                    "key":3,  
                    "value":"\u0000ð­ZÖÜ+?"  
                },  
                {  
                    "key":4,  
                    "value":"20230816"  
                }  
            ]  
        },  
        "upper_bounds":{  
            "array":[  
                {  
                    "key":1,  
                    "value":"õóÿÿÿÿÿÿ"  
                },  
                {  
                    "key":3,  
                    "value":"»`}[ºÿï?"  
                },  
                {  
                    "key":4,  
                    "value":"20230816"  
                }  
            ]  
        },  
        "key_metadata":null,  
        "split_offsets":{  
            "array":[  
                3  
            ]  
        },  
        "sort_order_id":{  
            "int":1  
        },  
        "index_files":{  
            "array":[  
                {  
                    "r146":{  
                        "index_id":{  
                            "int":1  
                        },  
                        "in_place":{  
                            "boolean":false  
                        },  
                        "index_data":{  
                            "string":"viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/index/log_date=20230816/00046-201-aad8de32-d5a9-4a47-a82e-bba9e83b1f3c-00001.orc_index_BLOOMFILTER_id.BLOOMFILTER"  
                        },  
                        "correlated_table_snapshot":{  
                            "long":-1  
                        }  
                    }  
                }  
            ]  
        },  
        "agg_index_files":{  
            "array":[  
  
            ]  
        },  
        "distribution_id":{  
            "int":1  
        }  
    }  
}

```

# meta文件数据
```
{
  "format-version" : 1,
  "table-uuid" : "05677643-62e7-42bb-83f1-522ec9031d6d",
  "location" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang",
  "last-updated-ms" : 1692701763566,
  "last-column-id" : 4,
  "schema" : {
    "type" : "struct",
    "schema-id" : 0,
    "fields" : [ {
      "id" : 1,
      "name" : "id",
      "required" : false,
      "type" : "long",
      "doc" : "测试"
    }, {
      "id" : 2,
      "name" : "data",
      "required" : false,
      "type" : "string",
      "doc" : "测试"
    }, {
      "id" : 3,
      "name" : "val",
      "required" : false,
      "type" : "double",
      "doc" : "测试"
    }, {
      "id" : 4,
      "name" : "log_date",
      "required" : false,
      "type" : "string",
      "doc" : "日期分区"
    } ]
  },
  "current-schema-id" : 0,
  "schemas" : [ {
    "type" : "struct",
    "schema-id" : 0,
    "fields" : [ {
      "id" : 1,
      "name" : "id",
      "required" : false,
      "type" : "long",
      "doc" : "测试"
    }, {
      "id" : 2,
      "name" : "data",
      "required" : false,
      "type" : "string",
      "doc" : "测试"
    }, {
      "id" : 3,
      "name" : "val",
      "required" : false,
      "type" : "double",
      "doc" : "测试"
    }, {
      "id" : 4,
      "name" : "log_date",
      "required" : false,
      "type" : "string",
      "doc" : "日期分区"
    } ]
  } ],
  "partition-spec" : [ {
    "name" : "log_date",
    "transform" : "identity",
    "source-id" : 4,
    "field-id" : 1000
  } ],
  "default-spec-id" : 0,
  "partition-specs" : [ {
    "spec-id" : 0,
    "fields" : [ {
      "name" : "log_date",
      "transform" : "identity",
      "source-id" : 4,
      "field-id" : 1000
    } ]
  } ],
  "last-partition-id" : 1000,
  "default-corr-cols-id" : 0,
  "last-corr-id" : 0,
  "corr-cols-specs" : [ {
    "spec-id" : 0,
    "corr-cols" : [ ]
  } ],
  "default-index-spec-id" : 0,
  "last-index-id" : 1,
  "index-specs" : [ {
    "index-spec-id" : 0,
    "index-fields" : [ ]
  }, {
    "index-spec-id" : 1,
    "index-fields" : [ {
      "index-id" : 1,
      "name" : "index_BLOOMFILTER_id",
      "type" : "BLOOMFILTER",
      "source-id" : 1,
      "transform" : "identity",
      "properties" : {
        "fpp" : "0.05"
      }
    } ]
  } ],
  "default-agg-index-spec-id" : 0,
  "last-agg-index-id" : 0,
  "agg-index-specs" : [ {
    "spec-id" : 0,
    "agg-indexes" : [ ]
  } ],
  "default-sort-order-id" : 0,
  "sort-orders" : [ {
    "order-id" : 0,
    "fields" : [ ]
  }, {
    "order-id" : 1,
    "fields" : [ {
      "transform" : "identity",
      "source-id" : 1,
      "direction" : "asc",
      "null-order" : "nulls-first"
    } ]
  } ],
  "default-distribution-id" : 0,
  "distributions" : [ {
    "distribution-id" : 0,
    "mode" : "none",
    "fields" : [ ]
  }, {
    "distribution-id" : 1,
    "mode" : "range",
    "fields" : [ {
      "transform" : "identity",
      "source-id" : 1
    } ]
  } ],
  "properties" : {
    "owner" : "hive",
    "external" : "true",
    "write.format.default" : "orc",
    "magnus.schedule.optimize.L0.file-num" : "10",
    "orc.compress" : "zstd",
    "comment" : "iceberg sdk数据写入测试用表",
    "write.target-file-size-bytes" : "67108864"
  },
  "current-snapshot-id" : 6178289509994994612,
  "snapshots" : [ {
    "snapshot-id" : 3926575333717895478,
    "parent-snapshot-id" : 7368050797715073152,
    "timestamp-ms" : 1692614031917,
    "summary" : {
      "operation" : "write_index",
      "added-data-files" : "136",
      "deleted-data-files" : "136",
      "added-records" : "1758400",
      "deleted-records" : "1758400",
      "added-files-size" : "9066125962",
      "removed-files-size" : "9066125962",
      "changed-partition-count" : "1",
      "total-records" : "1918203",
      "total-files-size" : "9079612489",
      "total-data-files" : "143",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/snap-3926575333717895478-1-c76750e0-06c2-4cf4-82aa-28b013b76efb.avro",
    "schema-id" : 0
  }, {
    "snapshot-id" : 6403747179647846330,
    "parent-snapshot-id" : 3926575333717895478,
    "timestamp-ms" : 1692652261280,
    "summary" : {
      "operation" : "replace",
      "manifests-created" : "1",
      "manifests-kept" : "0",
      "manifests-replaced" : "6",
      "entries-processed" : "0",
      "changed-partition-count" : "0",
      "total-records" : "1918203",
      "total-files-size" : "9079612489",
      "total-data-files" : "143",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/snap-6403747179647846330-1-f168eaeb-b124-44d5-8549-acdadea1621f.avro",
    "schema-id" : 0
  }, {
    "snapshot-id" : 7248291778686570806,
    "parent-snapshot-id" : 6403747179647846330,
    "timestamp-ms" : 1692699767062,
    "summary" : {
      "operation" : "append",
      "iceberg-sdk" : "trino@BILIBILI.CO",
      "added-data-files" : "1",
      "added-records" : "200",
      "added-files-size" : "10180",
      "changed-partition-count" : "1",
      "total-records" : "1918403",
      "total-files-size" : "9079622669",
      "total-data-files" : "144",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/snap-7248291778686570806-1-4efe2621-6cd1-48fb-af15-0e7763d3d991.avro",
    "schema-id" : 0
  }, {
    "snapshot-id" : 1350701407033072957,
    "parent-snapshot-id" : 7248291778686570806,
    "timestamp-ms" : 1692699816789,
    "summary" : {
      "operation" : "append",
      "iceberg-sdk" : "trino@BILIBILI.CO",
      "added-data-files" : "1",
      "added-records" : "200",
      "added-files-size" : "9921",
      "changed-partition-count" : "1",
      "total-records" : "1918603",
      "total-files-size" : "9079632590",
      "total-data-files" : "145",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/snap-1350701407033072957-1-026614b5-83d5-45f8-819d-e82a6c8e0d4e.avro",
    "schema-id" : 0
  }, {
    "snapshot-id" : 968701557809646354,
    "parent-snapshot-id" : 1350701407033072957,
    "timestamp-ms" : 1692699821295,
    "summary" : {
      "operation" : "append",
      "iceberg-sdk" : "trino@BILIBILI.CO",
      "added-data-files" : "1",
      "added-records" : "200",
      "added-files-size" : "9618",
      "changed-partition-count" : "1",
      "total-records" : "1918803",
      "total-files-size" : "9079642208",
      "total-data-files" : "146",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/snap-968701557809646354-1-bad7b2e8-9c63-4c5a-8fec-ecfe70556c4e.avro",
    "schema-id" : 0
  }, {
    "snapshot-id" : 6178289509994994612,
    "parent-snapshot-id" : 968701557809646354,
    "timestamp-ms" : 1692701763566,
    "summary" : {
      "operation" : "optimize",
      "spark.app.id" : "application_1691586627244_697272",
      "OPT_JOB_ID" : "1692701752728",
      "added-data-files" : "1",
      "deleted-data-files" : "2",
      "added-records" : "400",
      "deleted-records" : "400",
      "added-files-size" : "17824",
      "removed-files-size" : "19539",
      "changed-partition-count" : "1",
      "total-records" : "1918803",
      "total-files-size" : "9079640493",
      "total-data-files" : "145",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/snap-6178289509994994612-1-6ff3d923-707a-4e1f-8209-46f434ac0734.avro",
    "schema-id" : 0
  } ],
  "snapshot-log" : [ {
    "timestamp-ms" : 1692614031917,
    "snapshot-id" : 3926575333717895478
  }, {
    "timestamp-ms" : 1692652261280,
    "snapshot-id" : 6403747179647846330
  }, {
    "timestamp-ms" : 1692699767062,
    "snapshot-id" : 7248291778686570806
  }, {
    "timestamp-ms" : 1692699816789,
    "snapshot-id" : 1350701407033072957
  }, {
    "timestamp-ms" : 1692699821295,
    "snapshot-id" : 968701557809646354
  }, {
    "timestamp-ms" : 1692701763566,
    "snapshot-id" : 6178289509994994612
  } ],
  "metadata-log" : [ {
    "timestamp-ms" : 1692699522615,
    "metadata-file" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/00128-be4346a8-2a98-4203-a82c-c34d3d70b7d7.metadata.json"
  }, {
    "timestamp-ms" : 1692699559718,
    "metadata-file" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/00129-0c4a125b-fa61-4b62-8f1e-cea1c1e8700d.metadata.json"
  }, {
    "timestamp-ms" : 1692699767062,
    "metadata-file" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/00130-c5abeda5-731f-4f8d-b678-eb14f01c7423.metadata.json"
  }, {
    "timestamp-ms" : 1692699816789,
    "metadata-file" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/00131-19b47e54-57dd-472c-a370-99de9c38b11a.metadata.json"
  }, {
    "timestamp-ms" : 1692699821295,
    "metadata-file" : "viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/00132-914bb8df-18be-4af3-9b5a-180dd62cb7b0.metadata.json"
  } ],
  "column-dicts" : { }
}
```

# manifestList文件
```
{
    "manifest_path":"viewfs://jssz-bigdata-cluster/department/bigdata/test_olap_iceberg_warehouse/iceberg_sdk_test_yang/metadata/53a606a3-1725-489a-a6cb-edd9952d3999-m1.avro",
    "manifest_length":9975,
    "partition_spec_id":0,
    "added_snapshot_id":{
        "long":208439870123754921
    },
    "added_data_files_count":{
        "int":1
    },
    "existing_data_files_count":{
        "int":0
    },
    "deleted_data_files_count":{
        "int":0
    },
    "partitions":{
        "array":[
            {
                "contains_null":false,
                "contains_nan":{
                    "boolean":false
                },
                "lower_bound":{
                    "bytes":"20230820"
                },
                "upper_bound":{
                    "bytes":"20230820"
                }
            }
        ]
    },
    "added_rows_count":{
        "long":400
    },
    "existing_rows_count":{
        "long":0
    },
    "deleted_rows_count":{
        "long":0
    }
}
```
