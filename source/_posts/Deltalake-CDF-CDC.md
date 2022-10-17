---
title: Deltalake CDF & CDC
date: 2022-10-13 20:54:35
tags: delta
categories: 数据湖
cover: /img/topimg/16.png
---

## 目录格式
![delta_file.png](/img/blog/delta_file.png)

## CDC提交信息
![delta_metadata.png.png](/img/blog/delta_metadata.png)

> 注:以下元数据信息不是来自同一delta table,此处只是为了说明元数据包含的内容
### Create
```json
{"commitInfo":{"timestamp":1666008673760,"operation":"WRITE","operationParameters":{"mode":"ErrorIfExists","partitionBy":"[\"part\"]"},"isolationLevel":"Serializable","isBlindAppend":true,"operationMetrics":{"numFiles":"4","numOutputRows":"6","numOutputBytes":"2906"},"engineInfo":"Apache-Spark/3.3.0 Delta-Lake/2.2.0-SNAPSHOT","txnId":"c7bb9dc7-a4fd-4cab-a133-7a2981efc52e"}}
{"protocol":{"minReaderVersion":1,"minWriterVersion":4}}
{"metaData":{"id":"testId","format":{"provider":"parquet","options":{}},"schemaString":"{\"type\":\"struct\",\"fields\":[{\"name\":\"id\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}},{\"name\":\"text\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"part\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}}]}","partitionColumns":["part"],"configuration":{"delta.enableChangeDataFeed":"true"},"createdTime":1666008669098}}
{"add":{"path":"part=0/part-00000-45884e9c-0ad8-4eea-906e-1f2ad6ecee8b.c000.snappy.parquet","partitionValues":{"part":"0"},"size":748,"modificationTime":1666008673203,"dataChange":true,"stats":"{\"numRecords\":2,\"minValues\":{\"id\":0,\"text\":\"old\"},\"maxValues\":{\"id\":2,\"text\":\"old\"},\"nullCount\":{\"id\":0,\"text\":0}}"}}
{"add":{"path":"part=1/part-00000-71489cc6-540e-47d4-97b5-383d1b20dcc1.c000.snappy.parquet","partitionValues":{"part":"1"},"size":705,"modificationTime":1666008673475,"dataChange":true,"stats":"{\"numRecords\":1,\"minValues\":{\"id\":1,\"text\":\"old\"},\"maxValues\":{\"id\":1,\"text\":\"old\"},\"nullCount\":{\"id\":0,\"text\":0}}"}}
{"add":{"path":"part=0/part-00001-de6d02e2-23a8-44ea-b8c3-1ed6f99df4c1.c000.snappy.parquet","partitionValues":{"part":"0"},"size":705,"modificationTime":1666008673203,"dataChange":true,"stats":"{\"numRecords\":1,\"minValues\":{\"id\":4,\"text\":\"old\"},\"maxValues\":{\"id\":4,\"text\":\"old\"},\"nullCount\":{\"id\":0,\"text\":0}}"}}
{"add":{"path":"part=1/part-00001-9587e9a1-c653-4d41-9f1a-ed3e945c766a.c000.snappy.parquet","partitionValues":{"part":"1"},"size":748,"modificationTime":1666008673478,"dataChange":true,"stats":"{\"numRecords\":2,\"minValues\":{\"id\":3,\"text\":\"old\"},\"maxValues\":{\"id\":5,\"text\":\"old\"},\"nullCount\":{\"id\":0,\"text\":0}}"}}
```

### Insert
```json
{"add":{"path":"part-00000-488f0dec-025d-4f93-8ecd-b476c1fc491d-c000.snappy.parquet","partitionValues":{},"size":500,"modificationTime":1665587891400,"dataChange":true,"stats":"{\"numRecords\":5,\"minValues\":{\"id\":0},\"maxValues\":{\"id\":4},\"nullCount\":{\"id\":0}}"}}
{"add":{"path":"part-00001-3380b883-3558-498f-b8db-c054e17c88b5-c000.snappy.parquet","partitionValues":{},"size":503,"modificationTime":1665587891400,"dataChange":true,"stats":"{\"numRecords\":5,\"minValues\":{\"id\":5},\"maxValues\":{\"id\":9},\"nullCount\":{\"id\":0}}"}}
```

### Delete
```json
{"commitInfo":{"timestamp":1666008690517,"operation":"DELETE","operationParameters":{"predicate":"[\"(spark_catalog.delta.`C:\\\\Users\\\\Asura7969\\\\AppData\\\\Local\\\\Temp\\\\spark-5b6ca97d-30c8-4fa2-93da-2ff8e0872514`.part = 0L)\"]"},"readVersion":1,"isolationLevel":"Serializable","isBlindAppend":false,"operationMetrics":{"numRemovedFiles":"2","numAddedChangeFiles":"0","executionTimeMs":"592","scanTimeMs":"591","rewriteTimeMs":"0"},"engineInfo":"Apache-Spark/3.3.0 Delta-Lake/2.2.0-SNAPSHOT","txnId":"4784af1f-6d48-4fd4-8375-744414ddc76e"}}
{"remove":{"path":"part=0/part-00000-e9c04f85-a468-4646-9e1e-d7de0a0fd511.c000.snappy.parquet","deletionTimestamp":1666008689908,"dataChange":true,"extendedFileMetadata":true,"partitionValues":{"part":"0"},"size":965}}
{"remove":{"path":"part=0/part-00001-de6d02e2-23a8-44ea-b8c3-1ed6f99df4c1.c000.snappy.parquet","deletionTimestamp":1666008689908,"dataChange":true,"extendedFileMetadata":true,"partitionValues":{"part":"0"},"size":705}}
```

### Merge
```json
{"commitInfo":{"timestamp":1666008686732,"operation":"MERGE","operationParameters":{"predicate":"(s.id = t.id)","matchedPredicates":"[{\"predicate\":\"(s.id = 1L)\",\"actionType\":\"update\"},{\"predicate\":\"(s.id = 3L)\",\"actionType\":\"delete\"}]","notMatchedPredicates":"[]"},"readVersion":0,"isolationLevel":"Serializable","isBlindAppend":false,"operationMetrics":{"numTargetRowsCopied":"3","numTargetRowsDeleted":"1","numTargetFilesAdded":"2","executionTimeMs":"3631","numTargetRowsInserted":"0","scanTimeMs":"2294","numTargetRowsUpdated":"1","numOutputRows":"4","numTargetChangeFilesAdded":"1","numSourceRows":"4","numTargetFilesRemoved":"3","rewriteTimeMs":"1329"},"engineInfo":"Apache-Spark/3.3.0 Delta-Lake/2.2.0-SNAPSHOT","txnId":"cca2b832-199b-45bb-9606-9b83ab986c64"}}
{"remove":{"path":"part=1/part-00001-9587e9a1-c653-4d41-9f1a-ed3e945c766a.c000.snappy.parquet","deletionTimestamp":1666008686694,"dataChange":true,"extendedFileMetadata":true,"partitionValues":{"part":"1"},"size":748}}
{"remove":{"path":"part=1/part-00000-71489cc6-540e-47d4-97b5-383d1b20dcc1.c000.snappy.parquet","deletionTimestamp":1666008686731,"dataChange":true,"extendedFileMetadata":true,"partitionValues":{"part":"1"},"size":705}}
{"remove":{"path":"part=0/part-00000-45884e9c-0ad8-4eea-906e-1f2ad6ecee8b.c000.snappy.parquet","deletionTimestamp":1666008686731,"dataChange":true,"extendedFileMetadata":true,"partitionValues":{"part":"0"},"size":748}}
{"add":{"path":"part=0/part-00000-e9c04f85-a468-4646-9e1e-d7de0a0fd511.c000.snappy.parquet","partitionValues":{"part":"0"},"size":965,"modificationTime":1666008686629,"dataChange":true,"stats":"{\"numRecords\":2,\"minValues\":{\"id\":0,\"text\":\"old\"},\"maxValues\":{\"id\":2,\"text\":\"old\"},\"nullCount\":{\"id\":0,\"text\":0}}"}}
{"add":{"path":"part=1/part-00000-523f22c5-4a66-4889-ae47-37ca0b0574f2.c000.snappy.parquet","partitionValues":{"part":"1"},"size":927,"modificationTime":1666008686646,"dataChange":true,"stats":"{\"numRecords\":2,\"minValues\":{\"id\":1,\"text\":\"new\"},\"maxValues\":{\"id\":5,\"text\":\"old\"},\"nullCount\":{\"id\":0,\"text\":0}}"}}
{"cdc":{"path":"_change_data/part=1/cdc-00000-9cfe450c-00fe-460b-b384-3bba014467ab.c000.snappy.parquet","partitionValues":{"part":"1"},"size":1091,"dataChange":false}}
```

### Cdc
```json
{"commitInfo":{"timestamp":1665587909035,"operation":"Manual Update","operationParameters":{},"readVersion":1,"isolationLevel":"SnapshotIsolation","isBlindAppend":false,"operationMetrics":{},"engineInfo":"Apache-Spark/3.3.0 Delta-Lake/2.2.0-SNAPSHOT","txnId":"894b2f5f-b8bf-4b0b-89d6-84b9d9401409"}}
{"cdc":{"path":"part-00000-51911e8d-a7f6-46a5-bbbe-ca3bf0a76d8b-c000.snappy.parquet","partitionValues":{},"size":781,"dataChange":false}}
{"cdc":{"path":"part-00001-05ab5e6c-ac9d-43bd-ae91-3bb4555a1f88-c000.snappy.parquet","partitionValues":{},"size":786,"dataChange":false}}
```

cdc type:
* insert
* update_preimage
* update_postimage

| id  | _change_type     |
|-----|------------------|
| 20  | insert           |
| 21  | insert           |
| 22  | insert           |
| 23  | insert           |
| 24  | insert           |
| 26  | update_preimage  |
| 27  | update_postimage |

> 此表只有id一个字段, _change_type 为 delta内置字段


## 参考
> [阿里基于Delta Lake构建数据湖仓体系](https://bambrow.com/20210705-delta-lake-metadata-layout/)
> 
> [Delta Lake 表格底层数据结构初探](https://www.toutiao.com/article/7153161025360380457/?log_from=b2dd397da35ef_1665576765288)
