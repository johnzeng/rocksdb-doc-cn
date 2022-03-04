# 术语

**迭代器(Iterator)**： 迭代器被用户用于按顺序查询一个区间内的键值。参考
[https://rocksdb.org.cn/doc/Basic-Operations.html#iteration](https://rocksdb.org.cn/doc/Basic-Operations.html#iteration)

**点查询**：在RocksDB，点查询意味着通过 Get()读取一个键的值。

**区间查询**：区间查询意味着通过迭代器读取一个区间内的键值。

**SST文件（数据文件/SST表）**:SST是Sorted Sequence Table（排序队列表）。他们是排好序的数据文件。在这些文件里，所有键都按照排序好的顺序组织，一个键或者一个迭代位置可以通过二分查找进行定位。

**索引(Index)**： SST文件里的数据块索引。他会被保存成SST文件里的一个索引块。默认的索引个是是二分搜索索引。

**分区索引(Partitioned Index)**：被分割成许多更小的块的二分搜索索引。参考[https://rocksdb.org.cn/doc/Partitioned-Index-Filters.html](https://rocksdb.org.cn/doc/Partitioned-Index-Filters.html)

**LSM树**：参考定义：[https://en.wikipedia.org/wiki/Log-structured_merge-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)，RocksDB是一个基于LSM树的存储引擎

**前置写日志(WAL)或者日志(LOG)**：一个在RocksDB重启的时候，用于恢复没有刷入SST文件的数据的文件。参考 [https://rocksdb.org.cn/doc/Write-Ahead-Log-File-Format.html](https://rocksdb.org.cn/doc/Write-Ahead-Log-File-Format.html)

**memtable/写缓冲(write buffer)**：在内存中存储最新更新的数据的数据结构。通常它会按顺序组织，并且会包含一个二分查找索引。参考[https://rocksdb.org.cn/doc/Basic-Operations.html#memtable-and-table-factories](https://rocksdb.org.cn/doc/Basic-Operations.html#memtable-and-table-factories)

**memtable切换**：在这个过程，**当前活动的memtable**（现在正在写入的那个）被关闭，然后转换成 **不可修改memtable**。同时，我们会关闭当前的WAL文件，然后打开一个新的。

**不可修改memtable**：一个已经关闭的，正在等待被落盘的memtable。

**序列号(SeqNum/SeqNo)**：数据库的每个写入请求都会分配到一个自增长的ID数字。这个数字会跟键值对一起追加到WAL文件，memtable，以及SST文件。序列号用于实现snapshot读，压缩过程的垃圾回收，MVCC事务和其他一些目的。

**恢复**：在一次数据库崩溃或者关闭之后，重新启动的过程。

**落盘，刷新(flush)**：将memtable的数据写入SST文件的后台任务。

**压缩**：将一些SST文件合并成另外一些SST文件的后台任务。LevelDB的压缩还包括落盘。在RocksDB，我们进一步区分两个操作。查看[https://rocksdb.org.cn/doc/RocksDB-Basics.html#multi-threaded-compactions](https://rocksdb.org.cn/doc/RocksDB-Basics.html#multi-threaded-compactions)

**分层压缩或者基于分层的压缩方式**：RocksDB的默认压缩方式。

**全局压缩方式**：一种备选的压缩算法。参考 [https://rocksdb.org.cn/doc/Universal-Compaction.html](https://rocksdb.org.cn/doc/Universal-Compaction.html)

**比较器(comparator)**：一种插件类，用于定义键的顺序。参考[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/comparator.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/comparator.h)

**列族(column family)**：列族是一个DB里的独立键值空间。尽管他的名字有一定的误导性，他跟其他存储系统里的“列族”没有任何关系。RocksDB甚至没有“列”的概念。参考[https://rocksdb.org.cn/doc/Column-Families.html](https://rocksdb.org.cn/doc/Column-Families.html)

**快照(snapshot)**：一个快照是在一个运行中的数据库上的，在一个特定时间点上，逻辑一致的，视图。

**检查点(checkpoint)**：一个检查点是一个数据库在文件系统的另一个文件夹的物理镜像映射。参考 [https://rocksdb.org.cn/doc/Checkpoints.html](https://rocksdb.org.cn/doc/Checkpoints.html)

**备份(backup)**：RocksDB有一套备份工具用于帮助用户将数据库的当前状态备份到另一个地方，例如HDFS。参考[https://rocksdb.org.cn/doc/How-to-backup-RocksDB%3F.html](https://rocksdb.org.cn/doc/How-to-backup-RocksDB%3F.html)

**版本(Version)**：这个是RocksDB内部概念。一个版本包含某个时间点的所有存活SST文件。一旦一个落盘或者压缩完成，由于存活SST文件发生了变化，一个新的“版本”会被创建。一个旧的“版本”还会被仍在进行的读请求或者压缩工作使用。旧的版本最终会被回收。

**超级版本(super version)**：RocksDB的内部概念。一个超级版本包含一个特定时间的 的 一个SST文件列表（一个“版本”）以及一个存活memtable的列表。不管是压缩还是落盘，抑或是一个memtable切换，都会生成一个新的“超级版本”。一个旧的“超级版本”会被继续用于正在进行的读请求。旧的超级版本最终会在不再需要的时候被回收掉。

**块缓存(block cache)**：用于在内存缓存来自SST文件的热数据的数据结构。参考 [https://rocksdb.org.cn/doc/Block-Cache.html](https://rocksdb.org.cn/doc/Block-Cache.html)

**统计数据(statistics)**：一个在内存的，用于存储运行中数据库的累积统计信息的数据结构。[https://rocksdb.org.cn/doc/Statistics.html](https://rocksdb.org.cn/doc/Statistics.html)

**性能上下文(perf context)**：用于衡量本地线程情况的内存数据结构。通常被用于衡量单请求性能。参考 [https://rocksdb.org.cn/doc/Perf-Context-and-IO-Stats-Context.html](https://rocksdb.org.cn/doc/Perf-Context-and-IO-Stats-Context.html)

**DB属性（DB properties）**：一些可以通过DB::GetProperty()获得的运行中状态。

**表属性(table properites)**：每个SST文件的元数据。包括一些RocksDB生成的系统属性，以及一些通过用户定义回调函数生成的，用户定义的表属性。参考[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/table_properties.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/table_properties.h)

**写失速(write stall)**：如果有大量落盘以及压缩工作被积压，RocksDB可能会主动减慢写速度，确保落盘和压缩可以按时完成。参考[https://rocksdb.org.cn/doc/Write-Stalls.html](https://rocksdb.org.cn/doc/Write-Stalls.html)

**bloom filter**：参考[https://rocksdb.org.cn/doc/RocksDB-Bloom-Filter.html](https://rocksdb.org.cn/doc/RocksDB-Bloom-Filter.html)

**前缀bloom filter**：一种特殊的bloom filter，只能在迭代器里被使用。通过前缀提取器，如果某个SST文件或者memtable里面没有指定的前缀，那么可以避免这部分文件的读取。参考 [https://rocksdb.org.cn/doc/Prefix-Seek-API-Changes.html](https://rocksdb.org.cn/doc/Prefix-Seek-API-Changes.html)

**前缀提取器(prefix extractor)**：一个用于提取一个键的前缀部分的回调类。通常被用于前缀bloom filter。参考 [https://github.com/facebook/rocksdb/blob/master/include/rocksdb/slice_transform.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/slice_transform.h)

**基于块的bloom filter或者全bloom filter**： 两种不同的SST文件里的bloom filter存储方式。参考[https://rocksdb.org.cn/doc/RocksDB-Bloom-Filter.html#new-bloom-filter-format](https://rocksdb.org.cn/doc/RocksDB-Bloom-Filter.html#new-bloom-filter-format)

**分片过滤器(partitioned Filters)**：分片过滤器是将一个全bloom filter分片进更小的块里面，参考[https://rocksdb.org.cn/doc/Partitioned-Index-Filters.html](https://rocksdb.org.cn/doc/Partitioned-Index-Filters.html)

**压缩过滤器(compaction filter)**：一种用户插件，可以用于在压缩过程中，修改，或者丢弃一些键。参考[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/compaction_filter.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/compaction_filter.h)

**合并操作符(merge operator)**：RocksDB支持一种特殊的操作符Merge()，可以用于对现存数据进行差异纪录，合并操作。合并操作符是一个用户定义类，可以合并合并 操作。参考[https://rocksdb.org.cn/doc/Merge-Operator-Implementation.html](https://rocksdb.org.cn/doc/Merge-Operator-Implementation.html)

**基于块的表(block-based table)**：默认的SST文件格式。参考[https://rocksdb.org.cn/doc/Rocksdb-BlockBasedTable-Format.html](https://rocksdb.org.cn/doc/Rocksdb-BlockBasedTable-Format.html)

**块(block)**：SST文件的数据块。在SST文件里，一个块以压缩形式存储。

**平表(plain table)**：另一种SST文件格式，针对ramfs优化。参考[https://rocksdb.org.cn/doc/PlainTable-Format.html](https://rocksdb.org.cn/doc/PlainTable-Format.html)

**前向/反向迭代器**：一种特殊的迭代器选项，针对特定的使用场景进行优化。参考[https://rocksdb.org.cn/doc/Tailing-Iterator.html](https://rocksdb.org.cn/doc/Tailing-Iterator.html)。

**单点删除**：一种特殊的删除操作，只在用户从未更新过某个存在的键的时候被使用。参考：[https://rocksdb.org.cn/doc/Single-Delete.html](https://rocksdb.org.cn/doc/Single-Delete.html)

**限速器(rate limiter)**：用于限制落盘和压缩的时候写文件系统的速度。参考[https://rocksdb.org.cn/doc/Rate-Limiter.html](https://rocksdb.org.cn/doc/Rate-Limiter.html)

**悲观事务**：用锁来保证多个并行事务的独立性。默认的写策略是WriteCommited。

**两阶段提交(2PC,two phase commit)**：悲观事务可以通过两个阶段进行提交：先准备，然后正式提交。参考[https://rocksdb.org.cn/doc/Two-Phase-Commit-Implementation.html](https://rocksdb.org.cn/doc/Two-Phase-Commit-Implementation.html)

**提交写(WriteCommited)**：悲观事务的默认写策略，会把写入请求缓存在内存，然后在事务提交的时候才写入DB。

**预备写(WritePrepared)**：一种悲观事务的写策略，会把写请求缓存在内存，如果是二阶段提交，就在准备阶段写入DB，否则，在提交的时候写入DB。参考[https://rocksdb.org.cn/doc/WritePrepared-Transactions.html](https://rocksdb.org.cn/doc/WritePrepared-Transactions.html)

**未预备写(WriteUnprepared)**：一种悲观事务的写策略，由于这个是事务发送过来的请求，所以直接写入DB，以此避免写数据的时候需要使用过大的内存。


