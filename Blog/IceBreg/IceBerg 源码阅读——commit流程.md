该文章以SnapshotProducer和HadoopTableOperations解析[Iceberg](https://so.csdn.net/so/search?q=Iceberg&spm=1001.2101.3001.7020)中commit的流程，如有不足及错误之处，欢迎大家指正交流。

![[Picture/971aff09e62f848a4cc61573003bd3f2_MD5.png]]

首先，调用apply()获取最新的snapshot信息

1.  refresh()刷新tableMetadata
2.  获取当前snapshotId和下一个sequenceNumber
3.  validate(TableMetadata currentMetadata)校验操作是否有效
4.  apply(TableMetadata metadataToUpdate)更新到tableMetadata并返回新的manifest列表
5.  针对v2表或启用write.manifest-lists.enabled（默认为true，大部分都会写manifest-lists文件，即snap-xxx文件）
6.  manifestListPath()新建manifest-lists文件
7.  构建ManifestListWriter
8.  通过异步线程池遍历并缓存新的manifest列表
9.  通过ManifestListWriter将manifest信息写入manifestList

![[Picture/fc49f067cad174745d966ffd2099cc3d_MD5.png]]

随后，调用TableMetadata.buildFrom(base)构建update新的tableMetadata。并根据当前snapshot是否已存在于tableMetadata和是否支持WAP（write-audit-publish）[工作流](https://so.csdn.net/so/search?q=%E5%B7%A5%E4%BD%9C%E6%B5%81&spm=1001.2101.3001.7020)填充TableMetadata信息。构建新的tableMetadata，若元数据没有修改，则不需要commit

1.  若当前shapshot已存在于tableMetadata，则将新的snapshot置为main\_branch的snapshot
2.  若不满足1且支持WAP工作流，将新的snapshot添加到新的tableMetadata中
3.  如果都不满足，则将新的snapshot添加到新的tableMetadata中，并将新的snapshot置为main\_branch的snapshot

![[Picture/ebd3c0b8fb26355982c48003ec30585b_MD5.png]]

最后，调用TableOperations#commit(TableMetadata base, TableMetadata metadata)进行提交（以HadoopTableOperations为例）

1.  versionAndMetadata()获取当前的version和tableMetadata
2.  检查当前的tableMetadata与基准的tableMetadata相同，并新的tableMetadata与基准的tableMetadata相比有更新
3.  生成metadata临时目标文件路径，并将新的metadata内容写入该文件
4.  获取下一个version号，并生成最终metadata file
5.  将临时目标文件重命名为最终metadata文件
6.   将version信息写入VersionHint文件
7.  删除旧的metadata文件（将之前的metadata文件保存起来，在commit时检查下次不需要保留的进行移除）

![[Picture/84ca6241dbf5354cb75dd4f867c43283_MD5.png]]当确认commit成功之后，清理未commited的与未使用的manifest文件，并发送消息通知监听者。