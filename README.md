# rocksdb-doc-cn

这个是rocksdb文档的中文翻译

**严重推荐先阅读[FAQ](doc/RocksDB-FAQ.md)，能解决大部分问题**

原文来自[rocksdb wiki](https://github.com/facebook/rocksdb/wiki)

中文部分会更新到[rocksdb中文网/文档](https://rocksdb.org.cn/doc.html)

本翻译没有经过任何校对，so，如果有疑惑，欢迎提issue。如果希望帮助/催更翻译一些没有的章节，同样欢迎提issue。如果你自己翻译好了，提PR什么的也不是不可以。

目前我觉得比较有意思的内容已经翻一下来了，其他内容短时间内不会更新，如果希望看某一个篇章的翻译，请提issue。

# 目录

- [概述](doc/OverView.md)
- [FAQ](doc/RocksDB-FAQ.md)
- [术语](doc/Terminology.md) 
- 开发者指南
	- [基本操作](doc/Basic-Operations.md)
		- [迭代器](doc/iterator.md)
		- [前缀搜索](doc/Prefix-seek.md)
		- [向前搜索](doc/SeekForPrev.md)
		- [尾部迭代器](doc/Tailing-Iterator.md)
		- [读-修改-写操作符](doc/Merge-Operator.md)
        - [列族](doc/Column-Families.md)
        - [创建以及导入SST文件](doc/Creating-and-Ingesting-SST-files.md)
        - [单删除](doc/Single-Delete.md)
        - [低优先级写入](doc/Low-Priority-Write.md)
        - [生存时间(TTL)支持](doc/Time-to-Live.md)
        - [事务](doc/Transactions.md)
        - [快照](doc/Snapshot.md)
        - [范围删除](doc/DeleteRange.md)
        - [原子落盘](doc/Atomic-flush.md)
	- 选项
		- [基础选项以及调优](doc/Setup-Options-and-Basic-Tuning.md)
		- [选项字符串以及选项Map](doc/Option-String-and-Option-Map.md)
		- [配置文件](doc/RocksDB-Options-File.md)
    - [压缩/compression](doc/compression.md)
        - [字典压缩](doc/Dictionary-Compression.md)
    - [IO](doc/IO.md)
        - [限流器](doc/Rate-Limiter.md)
        - [直接IO](doc/Direct-IO.md)
    - [后台错误处理](doc/Background-Error-Handling.md)
    - [MANIFEST](doc/MANIFEST.md)
    - [块缓存](doc/Block-Cache.md)
    - [Memtable](doc/MemTable.md)
    - [巨型页帧支持](doc/Allocating-Some-Indexes-and-Bloom-Filters-using-Huge-Page-TLB.md)
    - [WAL日志](doc/Write-Ahead-Log.md)
        - [WAL日志格式](doc/Write-Ahead-Log-File-Format.md)
        - [WAL恢复模式](doc/WAL-Recovery-Modes.md)
    - [写缓冲管理器](doc/Write-Buffer-Manager.md)
    - [压缩/compaction](doc/Compaction.md)
        - [leveled-compaction](doc/Leveled-Compaction.md)
        - [universal-compaction](doc/Universal-Compaction.md)
        - [FIFO-compaction](doc/FIFO-compaction-style.md)
        - [手动压缩](doc/Manual-Compaction.md)
        - [子压缩](doc/Sub-Compaction.md)
    - [管理磁盘空间](doc/Managing-Disk-Space-Utilization.md)
    - SST文件格式
        - [基于块的表格式](doc/Rocksdb-BlockBasedTable-Format.md)
        - [平表](doc/PlainTable-Format.md)
        - [bloom过滤器](doc/RocksDB-Bloom-Filter.md)
        - [数据块哈希索引](doc/Data-Block-Hash-Index.md)
    - 日志以及监控
        - [日志](doc/Logger.md)
        - [统计](doc/Statistics.md)
        - [性能与IO上下文](doc/Perf-Context-and-IO-Stats-Context.md)
        - [事件监听器](doc/EventListener.md)
- 工具/实用助手
    - [checkpoint](doc/Checkpoints.md)
    - [如何备份RocksDB](doc/How-to-backup-RocksDB%3F.md)
- 实现细节
    - [删除过期文件](doc/Delete-Stale-Files.md)
    - [分片索引-过滤器](doc/Partitioned-Index-Filters.md)
    - [写预备事务](doc/WritePrepared-Transactions.md)
    - [写未预备事务](doc/WriteUnprepared-Transactions.md)
    - [我们是如何维护存活SST文件的](doc/How-we-keep-track-of-live-SST-files.md)
    - [优化SST文件索引以获得更好的搜索性能](doc/Indexing-SST-Files-for-Better-Lookup-Performance.md)
    - [合并运算实现](doc/Merge-Operator-Implementation.md)
    - [选择Level压缩的文件](doc/Choose-Level-Compaction-Files.md)
    - [RocksDB修复器](doc/RocksDB-Repairer.md)
    - [两步提交实现](doc/Two-Phase-Commit-Implementation.md)
    - [迭代器的实现](doc/Iterator-Implementation.md)
    - [模拟缓存](doc/Simulation-Cache.md)
    - [持久化读缓存](doc/Persistent-Read-Cache.md)
- RocksJava
    - [RocksJava基础](doc/RocksJava-Basics.md)
    - [RocksJava性能测试](doc/RocksJava-Performance-on-Flash-Storage.md)
- 性能
    - [RocksDB内存使用](doc/Memory-usage-in-RocksDB.md)
    - [调优指南](doc/RocksDB-Tuning-Guide.md)
    - [写失速](doc/Write-Stalls.md)
    - [使用RocksDB实现队列服务](doc/Implement-Queue-Service-Using-RocksDB.md)

# TODO

- 部分链接由于没有翻译，所以暂时没有编辑上去
- 校对。。。

