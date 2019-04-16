# 选项
除了按照[基础操作]()里的知道使用RocksDB写代码，你可能还会想知道如何调优RocksDB使得它的性能达到预期。在这一页，我们将介绍一个初始配置，这个配置在大多数场合都应该是能满足需求的。

RocksDB有非常多的配置，但是对于多数用户来说，很多选项都是可以不管的，因为他们里面的大多数，都只会影响特定的工作负荷。通常，大多数rocksDB的选项只要保持默认就好了，然而，针对以下这些选项，我们建议用户针对他们的工作负荷进行一定的试验。

首先，你需要关心资源限制相关的配置（同时参考 [基础操作]()）

## 写缓冲区大小 (Write Buffer Size)

这个可以对每个数据库和列族进行设置

### 列族写缓冲区大小
这个是一个列族可以用到的最大的缓冲区大小

它代表了用来存储写入磁盘前，没有排序过的内存数据的大小，默认值为64M。

你需要给这个值预留按照你的最坏内存使用情况  ＊ 2 的大小。如果你的内存不足，你应该减小这个值。否则，不推荐修改这个值。例如

cf_options.write_buffer_size = 64 << 20

参考下面关于列族间共享内存的配置

### 数据库写缓冲区大小

这是一个数据库里面所有跨列族写缓冲区内存的最大值。它代表了所有列族允许用于构建写盘前memtable的内存空间总大小。

这个功能默认是关闭的（设置为0）。一般不应该修改他。但是，为了举个例子，你确实需要设置他为64G的时候，可以这样做：

db_options.db_write_buffer_size = 64 << 30;

## 块缓存大小(Block Cache Size)

你可以按照你希望的大小创建一个块缓存，用于缓存没有压缩的数据。

我们推荐把这个数据设置为你可用内存的三分之一。剩下的空闲内存可以用于OS的页缓存。尽量将大块的内存留给操作系统有利于避免内存不足。（参考 [RocksDB内存使用]()）

设置块缓存大小同时需要我们针对表相关的内容进行设置，例如，如果你希望使用一个128M的LRU缓存，你应该这么做：

```cpp
auto cache = NewLRUCache(128 << 20);

BlockBasedTableOptions table_options;
table_options.block_cache = cache;

auto table_factory = new BlockBasedTableFactory(table_options);
cf_options.table_factory.reset(table_factory);

```

NOTE: 你应该对同一个进程的所有数据库的所有列族的table_options都使用同一个cache对象。为了达到这个目的，可以给所有列族和数据库的传递同一个tables_options或者table_factory。更多关于块存储的内容，参考 [块存储]()

## 压缩

你只能选择你的系统支持的压缩方式。使用压缩是使用CPU和IO换取存储空间

cf_options.compression控制前面n-1层的压缩方式。我们推荐使用LZ4(kLZ4Compression)，或者如果没有的话，使用Snappy（kSnappyCompression）

cf_options.bottonmost_compression控制第n层（也就是最后一层）的压缩方式。我们推荐使用ZStand(kZSTD)，或者，如果没有的话，使用Zlib（kZlibCompression）

了解更多压缩相关的内容，参考[压缩]()

## Bloom Filters

只有你确认这个符合你的查询模式的时候，才打开这个选项；如果你有点查询(Get())的需求，那么Bloom Filter可以加速这些请求，相反，如果你的查询大多数是区间扫描（例如Iterator()），那么Bloom Filter可能帮助不大

Bloom Filter会使用一个key里面的部分bit位，一个好的数字是10位，会带来大概1%的假阳性结果。

如果Get操作是一个常见操作，你可以配置Bloom Filter，例如，配置为使用10个bit位：

table_options.filter_policy.reset(NewBloomFilterPolicy(10, false));

想了解更多跟Bloom Filter有关的信息，可以参考 [Bloom Filter]()

## 速度限制

有时候限制压缩和写盘的速度，是优化IO的好办法，其中一个原因是，这样可以避免异常读延迟。可以通过设置db_options.rate_limiter选项来达到目的。限速是一个复杂的话题，将在 [速度限制章节]() 详细叙述

NOTE: 确保在一个进程的所有数据库中使用同一个rate_limiter对象。

## SST文件管理

如果你正在使用闪存存储，我们推荐你在挂在文件系统的时候打开discard开关，这样可以改善写放大。

如果你使用闪存存储并且打开了discard开关，那么自动整理就会开启。如果整理的空间比较大，自动整理可能会导致暂时的较大的IO延迟。SST文件管理可以控制文件删除速度，保证每次整理的空间大小是可控的。

SST管理文件可以通过设置db_options.sst_file_manager来打开。关于SST文件管理的细节，可以参考这里 [sst_file_manager_impl.h](https://github.com/facebook/rocksdb/blob/5.14.fb/util/sst_file_manager_impl.h#L28)

## 其他通用选项

下面这些选项，可以帮助我们在大多数情况获得一个合理的开箱即用的性能。由于担心用户在升级新版本的rocksdb时的兼容性以及性能恶化，我们一般不修改这些选项。我们建议用户在使用新的rocksdb工程的时候使用以下选项：

```cpp
cf_options.level_compaction_dynamic_level_bytes = true;
options.max_background_compactions = 4;
options.max_background_flushes = 2;
options.bytes_per_sync = 1048576;
options.compaction_pri = kMinOverlappingRatio;
table_options.block_size = 16 * 1024;
table_options.cache_index_and_filter_blocks = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;
```

如果你有服务使用默认选项运行，而不是使用这些设置，也不用灰心。尽管我们认为这些选项比默认选项好，但是它们一般也不会带来明显的性能优化。

## 结论以及延展阅读

现在你可以测试你的应用，然后看看你的初始化的RocksDB的性能了。希望他们足够好。

如果按照上面的设置，你的应用已经性能足够好了，我们不推荐你进一步进行调优。由于工作压力经常变化，如果你为了提高当前工作压力下的RocksDB性能，而增加了一些无用的资源，有些不起眼的设置，也可能在未来的工作压力下导致RocksDB雪崩。

另一方面，如果性能不能满足你的要求，你可以根据 [调优指南]()进一步调优RocksDB。


