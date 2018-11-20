
**问:如果我的进程crash了，我的数据库数据会受影响吗？**
答：不悔，但是如果你没有开启WAL没有刷入到存储介质的memtable数据可能会丢失。

**问:如果我的机器crash了，RocksDB能保证数据的完整吗？**
答：数据在你调用一个带sync的写请求的时候会被写入磁盘（使用WriteOptions.sync=true的写请求），或者你可以等待memtable被刷入存储的时候。

**问:RocksDB会抛异常嘛？**
答：不，出错的时候RocksDB会返回一个rocksdb::Status，用于指明错误。然后RocksDB本身也不捕捉来自STL库和其他依赖的异常，所以如果内存不够，你可能会遇到std::bad_malloc异常，或者类似的场景下的类似的异常。

**问:如何获取RocksDB里面存储的键值的数量？**
答：使用GetIntProperty(cf_handle, “rocksdb.estimate-num-keys") 可以获得存储在一个列族里的键的大概数量。GetAggregatedIntProperty(“rocksdb.estimate-num-keys", &num_keys) 可以获得整个RocksDB数据库的键数量。

**问:为什么GetIntProperty只能返回RocksDB总键数的估算值？**
答：在像RocksDB这种LSM数据库里获得key的总数量是一个非常有挑战性的问题，因为他们总是存储有一些重复的key和一些被删除的项目，所以如果你需要精确数量，你需要进行一次全压缩。另外，如果RocksDB的数据库使用了合并操作符，这个估算的值将更加不准确。

**问:哪些基本操作，Put(),Write(),NewIterator()，是线程安全的吗**
答：是的

**问:我可以从多个进程同时写RocksDB吗？**
答：不可以。然而，你可以在多个进程中用只读模式打开RocksDB

**问:RocksDB支持多进程读吗？**
答：RocksDB支持多个进程以只读模式打开rocksDB，可以通过使用DB::OpenForReadOnly()打开数据库。

**问:最大支持的key和value的大小是多少**
答：RocksDB不是针对大key设计的。推荐的最大key value大小分别为8MB和3GB

**问:我可以在HDFS上跑RocksDB吗？**
答：可以，使用NewHdfsEnv()接口返回的Env，RocksDB就可以把数据存储到HDFS上了。然而HDFS上目前无法支持文件锁。

**问:我可以保存一个snapshot，然后回滚rocksDB到这个状态吗？**
答：可以，请使用BackUpEngine或则Checkpoints接口

**问:如何快速将数据载入rocksdb**
答：其中一种方法是直接往RocksDB里面插入数据：

- 使用单一写线程，然后按顺序插入
- 将数百个键值在一个写请求插入
- 使用vector memtable
- 确保options.max_background_flushes至少为4
- 开始插入数据前，关闭自动压缩，将options.level0_file_num_compaction_trigger, options.level0_slowdown_writes_trigger and options.level0_stop_writes_trigger 设置到非常大的值。插入完成后，开始一次手动压缩。
	
3-5条可以通过调用Options::PrepareForBulkLoad()一次完成。

如果你可以离线处理数据，有一个更加快的方法：你可以对数据进行排序，并行生成没有交集的SST文件，然后批量加载这些SST文件接口。参考 [创建以及导入SST文件]()

**问: RocksJava已经支持全部功能了嘛？**
答：我们还在开发RocksJava。当然，如果你发现某些功能缺失，你也可以主动提交PR。

**问:谁在用RocksDB**
答：参考 [Users]()

**问:如何正确删除一个DB？我可以直接对活动的DB调用DestoryDB吗？**
答：先调用close然后调用destory才是正确的方法。对一个活动的DB调用DestoryDB会导致未定义行为。

**问:调用DestoryDB和直接删除DB所在的文件夹有什么差别？**
答：如果你的数据存放在多个文件夹，DestoryDB会处理好这些文件夹。一个DB可以通过配置DBOptions::db_paths, DBOptions::db_log_dir, 和 DBOptions::wal_dir以将数据存储在不同的目录。

**问:BackupableDB会创建一个指定时间的snapshot吗？**
答：调用CreateNewBackup的时候，如果BackupOptions::backup_log_files = true或者flush_before_backup为true，就会创建。

**问:备份进程会影响到其他数据库访问吗？**
答：不会，你可以在备份的同时读写数据库。

**问:有什么更好的办法把map-reduce生成的数据导入到rocksDB吗？**
答：使用SstFileWriter，它可以让你直接创建RocksDB的SST文件，然后直接把他们加入到数据库。然而，如果你把SST文件加入到已有的RocksDB数据库，那么这些键不能跟已有的键值有交集。参看 [创建以及导入SST文件]()

**问:对不同的列族配置不同的前缀提取器安全吗？**
答：安全。

**问:RocksDB编译需要的最小gcc版本是多少？**
答：可以使用4.7。但是推荐4.8及以上的。

**问:如何配置RocksDB以使用多个磁盘？**
答：你可以在多个磁盘上创建一个文件系统（ext3，xfs，等）。然后你就可以在这个单一文件系统上跑RocksDB了。使用多个磁盘的一些tips：
- 如果使用RAID，请使用大的stripe size（64kb太小，推荐使用1MB）
- 考虑把ColumnFamilyOptions::compaction_readahead_size设置到大于2MB以打开压缩预读
- 如果主要的工作压力是写，可以增加压缩线程以保持磁盘满负载工作。
- 考虑为压缩打开一步写

**问:直接拷贝一个打开的RocksDB实例安全吗？**
答：不安全，除非这个实例是以只读模式打开。

**问:如果我用一种新的压缩方式打开RocksDB，我还能读到旧的（用其他压缩方式保存的）数据吗？**
答：可以，RocksDB会在SST文件里面保存压缩方式，所以就算换了压缩方式，仍旧能读到现有的文件。你甚至可以通过ColumnFamilyOptions::bottommost_compression给最底层换一个压缩算法。

**问:我想在HDFS上备份RocksDB，怎么配置呢？**
答：使用BackupableDB然后把backup_env设置为NewHdfsEnv()的返回值即可。

**问:在压缩过滤器的回调里面读，写rocksDB是不是安全的？**
答：读是安全的，但是写不一定总是安全的，因为在要锁过滤器的回调里写数据可能会在触发停写条件的时候造成死锁。

**问:生成snapshot 的时候，RocksDB会保留SST文件以及memtable吗？**
答：不悔，参考 [snapshot 的工作原理]()

**问:如果设置了DBWithTTL，过期的key被删除的时间有保证吗？**
答：没有，DBWithTTL不保证一个时间上限。过期的key只在他们被压缩的时候被删除。然而，没人保证这个压缩一定会发生。例如，你有一部分key永远都不会更新，那么压缩基本不会影响这部分key所以他们也不会因过期而被删除。
**问:**
答：


