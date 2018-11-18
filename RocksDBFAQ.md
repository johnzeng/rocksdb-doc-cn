**问:**
答：

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





