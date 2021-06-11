这里我们尝试解释Rocksdb如何使用内存。Rocksdb中有几个组件会贡献内存使用：

- 块缓存(Block cache)
- 索引和bloom过滤器
- Memtable
- 迭代器固定的块

我们会轮流讨论他们

# 块缓存

块缓存是Rocksdb缓存未压缩数据块的地方。你可以通过BlockBasedTableOptions的block_cache配置块缓存的大小：

```
rocksdb::BlockBasedTableOptions table_options;
table_options.block_cache = rocksdb::NewLRUCache(1 * 1024 * 1024 * 1024LL);
rocksdb::Options options;
options.table_factory.reset(new rocksdb::BlockBasedTableFactory(table_options));
```

如果数据块在块缓存中没有被发现，Rocksdb使用带缓存的IO从文件中读取他。这意味着他也会使用页缓存 —— 这里包含了原始的压缩过的块。这样，Rocksdb的缓存是两层的：块缓存和页缓存。反直觉的一件事情是，减小块缓存不会增加IO。存起来的内存可能会被用于页缓存，所以会有更多的数据被缓存起来。然而，CPU使用量可能会增加，因为Rocksdb需要解压他从页缓存读取的数据。

为了学习块缓存使用的量有多少，你可以在一个块缓存对象上调用GetUsage()：

```
table_options.block_cache->GetUsage();
```

在MongoRocks，你可以通过调用以下命令得到块缓存大小：

```
> db.serverStatus()["rocksdb"]["block-cache-usage"]
```

# 索引和过滤块

索引和过滤块可能是内存使用大户，并且默认他们不会计算在你分配给块缓存的内存里。这有时候会迷惑用户：你分配了10GB给块缓存，但是Rocksdb使用了15GB的内存。这个差异通常是因为内存和bloom过滤块。

这里介绍你如何大致计算和管理索引和过滤块的大小：

	对于每个数据块我们存储三个信息到索引：一个key，一个偏移，以及一个大小。因此有两个方法你可以减少索引的大小。
    如果你增加块大小，块的数量会减少，所以索引的大小会线性减少。
    默认情况下我们的块大小为4KB，尽管我们通常在生产环境使用16~32KB。第二个减少索引大小的方法是减少key的大小，尽管对于某些使用场景，这不现实。
	计算过滤块的大小的方法很简单。如果你配置bloom过滤器为每个key取10个bit（默认，会有1%的假阳性），bloom过滤器大小为number_of_keys * 10 bit。
    不过你可以使用一个技巧。如果你确认Get()在最坏的情况下都能找到一个key，你可以设置options.optimize_filters_for_hits为true。
    打开了这个选项，我们会不再最后一层，包含90%的数据库内容，构建bloom过滤器。因此，内存的bloom过滤器的使用量会少10X倍。
    但是，对于不能找到数据的Get请求，你每次都要花费一个额外的IO。
	
有两个选项用于配置索引和过滤块使用多少内存的：

	如果你设置cache_index_and_filter_blocks为true，索引和过滤块会被存储在块缓存，跟其他数据块一起。
    这同时意味着他们会被页换出。如果你的访问方式局部性比较强（比如，你有一些非常少用的key范围），这个设置就很合理了。
    然而，在大多数情况，他都是对你的性能有害的，因为你需要索引和过滤器来访问一个特定的文件。
    如果你确定你的ulimit总是大于数据库中的文件数量，我们推荐你设置max_open_files为-1，以为这无限制。
    这个选项会预加载所有过滤器和索引块，并且不需要维护文件的LRU。设置max_open_files为-1会给你最好的性能。
	
	为了了解索引和过滤块使用的内存，你可以使用RocksDB的GetProperty() API：
	
```
std::string out;
db->GetProperty("rocksdb.estimate-table-readers-mem", &out);
```

在MongoRock，只需要在mongo shell调用这个API：

```
> db.serverStatus()["rocksdb"]["estimate-table-readers-mem"]
```

在[分片索引/过滤](Partitioned-Index-Filters.md)中，分片总是存储在块缓存。顶级索引可以通过cache_index_and_filter_blocks被配置为存储在堆或者块缓存。

# Memtable

你可以认为memtable是内存写缓冲。每个新的键值对先写入到memtable。Memtable大小通过选项write_buffer_size控制。通常他不是内存消耗大户。然而，memtable的大小与写放大成反比 —— 你给memtable的内存越多，写放大越小。如果你增加你的memtable的大小，确保增加你的L1的大小，L1的大小通过选项max_bytes_for_level_base控制。

为了获得当前的memtable大小，你可以使用：

```
std::string out;
db->GetProperty("rocksdb.cur-size-all-mem-tables", &out);
```

在MongoRocks，等价的调用是：

```
> db.serverStatus()["rocksdb"]["cur-size-all-mem-tables"]
```

从5.6版本开始，你可以把memtable放入块缓存中。参考[Write-Buffer-Manager](Write-Buffer-Manager.md)。

# 迭代器固定的块

迭代器固定的块通常不会贡献太多的内存使用量。然而，在某些时候，当你有100k读事务同步发生，这就会给内存造成压力了。固定的块的内存使用很容易计算。每个迭代器固定L0的每个文件的一个数据块，加上每个L1+level的一个数据块。所以迭代器固定的块的总的内存使用接近 num_iterators * block_size * ((num_levels-1) + num_l0_files)。为了获得这个内存使用的统计信息，调用在块缓存对象上调用GetPinnedUsage()：

```
 table_options.block_cache->GetPinnedUsage();
```


