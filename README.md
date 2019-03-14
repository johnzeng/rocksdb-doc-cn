# rocksdb-doc-cn

这个是rocksdb文档的中文翻译


# 目录

- [概述](OverView.md)
- [FAQ](RocksDBFAQ.md)
- [术语](Terminology.md) 
- 开发者指南
	- [基本操作](basic_operation.md)
		- [迭代器](iterator.md)
		- [前缀搜索](Prefix-seek.md)
		- [向前搜索](SeekForPrev.md)
		- [尾部迭代器](tailing-iteration.md)
		- [读-修改-写操作符](Merge-operator.md)
        - [列族](Column-Families.md)
        - [创建以及导入SST文件](Creating-and-Ingesting-SST-files.md)
        - [单删除](Single-Delete.md)
        - [低优先级写入](low-priority-write.md)
        - [生存时间(TTL)支持](Time-to-Live.md)
        - [事务](Transactions.md)
        - [快照](Snapshot.md)
	- 选项
		- [基础选项以及调优](Setup-Options-and-Basic-Tuning.md)
		- [选项字符串以及选项Map](Option-String-and-Option-Map.md)
		- [配置文件](RocksDB-Options-File.md)
    - [压缩/compression](compression.md)
        - [字典压缩](Dictionary-Compression.md)
    - [IO](IO.md)
        - [限流器](rate-limiter.md)
        - [直接IO](direct-io.md)
    - [后台错误处理](background-error-handling.md)
    - [MANIFEST](MANIFEST.md)
    - [块缓存](Block-Cache.md)
    - [Memtable](MemTable.md)
    - [巨型页帧支持](Allocating-Some-Indexes-and-Bloom-Filters-using-Huge-Page-TLB.md)
    - [WAL日志](Write-Ahead-Log.md)
        - [WAL日志格式](Write-Ahead-Log-File-Format.md)
        - [WAL恢复模式](WAL-Recovery-Modes.md)
    - [写缓冲管理器](Write-Buffer-Manager.md)
    - [压缩/compaction](Compaction.md)
        - [leveled-compaction](Leveled-Compaction.md)
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
- RocksJava
    - [RocksJava基础](RocksJava-Basics.md)
    - [RocksJava性能测试](RocksJava-Performance-on-Flash-Storage.md)

# TODO

- 目前目录中还没翻译的内容
- 部分链接由于没有翻译，所以暂时没有编辑上去
- 校对。。。

