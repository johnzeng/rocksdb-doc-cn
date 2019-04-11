# rocksdb-doc-cn

这个是rocksdb文档的中文翻译

**严重推荐先阅读FAQ，解决大部分问题**

原文来自[rocksdb wiki](https://github.com/facebook/rocksdb/wiki)

中文部分会更新到[rocksdb中文网/文档](https://rocksdb.org.cn/doc.html)

本翻译没有经过任何校对，so，如果有疑惑，欢迎提issue。如果希望帮助/催更翻译一些没有的章节，同样欢迎提issue。如果你自己翻译好了，提PR什么的也不是不可以。

目前翻译中的章节我会在下方的TODO更新。

# 目录

- [概述](OverView.md)
- [FAQ](RocksDBFAQ.md)
- [术语](Terminology.md) 
- 开发者指南
	- [基本操作](Basic-Operations.md)
		- [迭代器](iterator.md)
		- [前缀搜索](Prefix-seek.md)
		- [向前搜索](SeekForPrev.md)
		- [尾部迭代器](Tailing-Iterator.md)
		- [读-修改-写操作符](Merge-operator.md)
        - [列族](Column-Families.md)
        - [创建以及导入SST文件](Creating-and-Ingesting-SST-files.md)
        - [单删除](Single-Delete.md)
        - [低优先级写入](low-priority-write.md)
        - [生存时间(TTL)支持](Time-to-Live.md)
        - [事务](Transactions.md)
        - [快照](Snapshot.md)
        - [范围删除](DeleteRange.md)
        - [原子落盘](Atomic-flush.md)
	- 选项
		- [基础选项以及调优](Setup-Options-and-Basic-Tuning.md)
		- [选项字符串以及选项Map](Option-String-and-Option-Map.md)
		- [配置文件](RocksDB-Options-File.md)
    - [压缩/compression](compression.md)
        - [字典压缩](Dictionary-Compression.md)
    - [IO](IO.md)
        - [限流器](Rate-Limiter.md)
        - [直接IO](Direct-IO.md)
    - [后台错误处理](Background-Error-Handling.md)
    - [MANIFEST](**MANIFEST.md**)
    - [块缓存](Block-Cache.md)
    - [Memtable](**MemTable.md**)
    - [巨型页帧支持](Allocating-Some-Indexes-and-Bloom-Filters-using-Huge-Page-TLB.md)
    - [WAL日志](**Write-Ahead-Log.md**)
        - [WAL日志格式](Write-Ahead-Log-File-Format.md)
        - [WAL恢复模式](WAL-Recovery-Modes.md)
    - [写缓冲管理器](Write-Buffer-Manager.md)
    - [压缩/compaction](Compaction.md)
        - [leveled-compaction](**Leveled-Compaction.md**)
        - [universal-compaction](Universal-Compaction.md)
        - [FIFO-compaction](FIFO-compaction-style.md)
        - [手动压缩](Manual-Compaction.md)
        - [子压缩](Sub-Compaction.md)
    - [管理磁盘空间](Managing-Disk-Space-Utilization.md)
    - SST文件格式
        - [基于块的表格式](Rocksdb-BlockBasedTable-Format.md)
        - [平表](PlainTable-Format.md)
        - [bloom过滤器](RocksDB-Bloom-Filter.md)
        - [数据块哈希索引](Data-Block-Hash-Index.md)
    - 日志以及监控
        - [日志](Logger.md)
        - [统计](Statistics.md)
        - [性能与IO上下文](Perf-Context-and-IO-Stats-Context.md)
        - [事件监听器](EventListener.md)
- 工具/实用助手
    - [checkpoint](Checkpoints.md)
- 实现细节
    - [删除过期文件](Delete-Stale-Files.md)
    - [分片索引-过滤器](Partitioned-Index-Filters.md)
    - [写预备事务](WritePrepared-Transactions.md)
    - [写未预备事务](WriteUnprepared-Transactions.md)
    - [我们是如何维护存活文件的](How-we-keep-track-of-live-SST-files.md)
    - [优化SST文件索引](Indexing-SST-Files-for-Better-Lookup-Performance.md)
    - [选择Level压缩的文件](Choose-Level-Compaction-Files.md)
    - [RocksDB修复器](RocksDB-Repairer.md)
    - [迭代器的实现](Iterator-Implementation.md)
    - [模拟内存](Simulation-Cache.md)
    - [持久化读缓存](Persistent-Read-Cache.md)
- RocksJava
    - [RocksJava基础](RocksJava-Basics.md)
    - [RocksJava性能测试](RocksJava-Performance-on-Flash-Storage.md)

# TODO

- Implementation Details好评翻译中
- 部分链接由于没有翻译，所以暂时没有编辑上去
- 校对。。。

