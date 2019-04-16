块缓存是Rocksdb在内存中缓存数据以用于读取的地方。用户可以带上一个期望的空间大小，传一个Cache对象给Rocksdb实例。一个缓存对象可以在同一个进程的多个RocksDB实例之间共享，这允许用户控制总的缓存大小。块缓存存储未压缩过的块。用户也可以选择设置另一个块缓存，用来存储压缩后的块。读取的时候会先拉去未压缩的数据块的缓存，然后才拉取压缩数据块的缓存。在打开直接IO的时候压缩块缓存可以替代OS的页缓存。

RocksDB里面有两种实现方式，分别叫做LRUCache和ClockCache。两个类型的缓存都通过分片来减轻锁冲突。容量会被平均的分配到每个分片，分片之间不共享空间。默认情况下，每个缓存会被分片到64个分片，每个分片至少有512kB空间。

# 使用

开箱即用的情况下，RocksDB会使用LRU块缓存实现，空间为8MB。如果希望使用自定义的块缓存，调用N额外LRUCache()或者NewClockCache()来创建一个缓存对象，然后把它设置到基于块的表选项。用户也可以使用自己实现的缓存，只需要实现Cache接口即可

```cpp
std::shared_ptr<Cache> cache = NewLRUCache(capacity);
BlockBasedTableOptions table_options;
table_options.block_cache = cache;
Options options;
options.table_factory.reset(new BlockBasedTableFactory(table_options));
```

如果希望设置压缩块缓存
```cpp
table_options.block_cache_compressed = another_cache;
```

如果block_cache是空指针，RocksDB会创建默认的块缓存。如果希望彻底关闭块缓存，你需要：

```cpp
table_options.no_block_cache = true;
```

# LRU缓存

开箱即用的情况下，RocksDB会使用LRU块缓存实现，空间为8MB。每个缓存分片都维护自己的LRU列表以及自己的查找哈希表。通过每个分片持有一个互斥锁来实现并发。不管是查找还是插入，都需要申请该分片的互斥锁。用户可以通过调用NewLRUCache创建一个LRU缓存。函数提供了一些非常有用的选项来设置缓存：

- capacity: 缓存的总大小
- num_shar_bits：该缓存需要提取key的多少个bit来生成分片id。缓存会被分片成2^num_shard_bits个分片。
- strict_capacity_limit：在非常少有的情况下，缓存块大小会比他的容量大。比如说，正在进行的读取操作或者整个DB的迭代器遍历操作把块钉在了块缓存里，然后被固定的块的大小超过了容量。如果后面有读取操作，希望将数据插入到块缓存中，如果strict_capacity_limit=false（这是默认的），缓存会没法按照容量设置进行处理，只能同意插入。如果主机内存不够，这可能导致未预期的OOM错误导致DB崩溃。把这个选项设置为true会拒绝进一步的缓存插入，并且导致读或者迭代失败。这个选项是按照分片粒度来工作的，也就是说有可能一个分片满的时候拒绝插入，而其他分片还有未使用的空间。
- high_pri_pool_ratio：保留给高优先级块的容量的百分比。参考[缓存索引与过滤块]()章节了解更多内容

# Clock缓存

ClockCache实现了[CLOCK算法](https://en.wikipedia.org/wiki/Page_replacement_algorithm#Clock)。每个clock缓存分片都维护一个缓存项的环形列表。一个clock指针遍历这个环形列表来找一个没有固定的项进行驱逐，同时，如果在上一个扫描中他被使用过了，那么给予这个项两次机会来留在缓存里。tbb::concurrent_hash_map 被用来查找数据。

与LRU缓存比较，clock缓存有更好的锁粒度。在LRU缓存下面，每个分片的互斥锁在读取的时候都需要上锁，因为他需要更新他的LRU列表。在一个clock缓存上查找数据不需要申请该分片的互斥锁，只需要搜索并行的哈希表就行了，所以有更好锁粒度。只有在插入的时候需要每个分片的锁。用clock缓存，在一定环境下，我们能看到读性能的增长。（参考cache/clock_cache.cc的注释以了解这个性能测试的设置）

```
Threads Cache     Cache               ClockCache               LRUCache
        Size  Index/Filter Throughput(MB/s)   Hit Throughput(MB/s)    Hit
    32   2GB       yes               466.7  85.9%           433.7   86.5%
    32   2GB       no                529.9  72.7%           532.7   73.9%
    32  64GB       yes               649.9  99.9%           507.9   99.9%
    32  64GB       no                740.4  99.9%           662.8   99.9%
    16   2GB       yes               278.4  85.9%           283.4   86.5%
    16   2GB       no                318.6  72.7%           335.8   73.9%
    16  64GB       yes               391.9  99.9%           353.3   99.9%
    16  64GB       no                433.8  99.8%           419.4   99.8%
```

如果需要构造一个clock缓存，调用NewClockCache，如果需要使用clock缓存，RocksDB需要与[intel TBB库](https://www.threadingbuildingblocks.org/)链接。clock缓存在构建的时候也有几个选项供用户选择：

- capacity： 与LRUCache相同
- num_shard_bits： 与LRUCache相同
- strict_capacity_limit： 与LRUCache相同

# 缓存索引以及过滤块

默认情况下，索引和过滤块都在块缓存外面存储，并且，除了max_open_files，用户无法控制使用多少内存来缓存这些块。用户可以选择在块缓存中缓存索引和过滤块，这样可以更好的控制RocksDB的缓存。在块缓存中缓存索引和过滤块：

```cpp
BlockBasedTableOptions table_options;
table_options.cache_index_and_filter_blocks = true;

```

通过把索引和过滤块放入块缓存，这些块需要和数据块竞争来留存在缓存中。尽管索引和过滤块的访问频率比数据块高，也有一些情况会出现相反的情况。这不是预期行为，因为索引和过滤块一般会比数据块更大，并且他们通常留存在内存的价值更大。有两个选项用来调优减轻这些问题：

- cache_index_and_filter_blocks_with_high_priority ： 把块缓存中的索引和过滤块缓存设置为高优先级。目前这个选项只对LRUCcache有用，并且需要调用NewLRUCache的时候配合high_pri_pool_ratio使用。如果这个功能打开，LRU缓存中的LRU列表会分成两个部分，一个用于高优先级块，一个用于低优先级块。数据块会被插入到低优先级的池的头部。索引和过滤器块会被插入到高优先级池的头部。如果高优先级池的总大小超过了capacity * high_pri_pool_ratio，高优先级池的块会溢出到低优先级池，然后他开始跟低优先级池的数据发生竞争。淘汰会从低优先级池开始。
- pin_l0_filter_and_index_blocks_in_cache：把level0的文件的索引和过滤块钉在块缓存中，避免他们被淘汰。level0的索引和过滤块通常会非常频繁的被访问。而且通常他们也比较小，所以通常来说他们不会浪费太多空间

# 模拟缓存

SimCache是一个工具，用来在缓存容量和数量变化的时候预测缓存命中率。他通过封装DB正在使用的真正的Cache对象，然后根据给定的容量和分片大小，运行一个后台的LRU缓存模拟，以测量缓存命中率和影子缓存的丢失率。这个工具在用户希望打开一个，比如说，有4GB的缓存大小的DB，但是希望知道如果缓存空间扩展到，比如说，64GB，的时候，缓存命中率会是多少？创建一个模拟缓存：

```cpp
// This cache is the actual cache use by the DB.
std::shared_ptr<Cache> cache = NewLRUCache(capacity);
// This is the simulated cache.
std::shared_ptr<Cache> sim_cache = NewSimCache(cache, sim_capacity, sim_num_shard_bits);
BlockBasedTableOptions table_options;
table_options.block_cache = sim_cache;
```

模拟缓存需要的额外数据量小于sim_capacity的2%

# 统计

如果Options.statistics不是null，那么会有一系列的块缓存计数器可以被访问

```
// 总块缓存不明中数
// REQUIRES: BLOCK_CACHE_MISS == BLOCK_CACHE_INDEX_MISS +
//                               BLOCK_CACHE_FILTER_MISS +
//                               BLOCK_CACHE_DATA_MISS;
BLOCK_CACHE_MISS = 0,
// 总块缓存命中
// REQUIRES: BLOCK_CACHE_HIT == BLOCK_CACHE_INDEX_HIT +
//                              BLOCK_CACHE_FILTER_HIT +
//                              BLOCK_CACHE_DATA_HIT;
BLOCK_CACHE_HIT,
// # of blocks added to block cache.
BLOCK_CACHE_ADD,
// # of failures when adding blocks to block cache.
BLOCK_CACHE_ADD_FAILURES,
// # of times cache miss when accessing index block from block cache.
BLOCK_CACHE_INDEX_MISS,
// # of times cache hit when accessing index block from block cache.
BLOCK_CACHE_INDEX_HIT,
// # of times cache miss when accessing filter block from block cache.
BLOCK_CACHE_FILTER_MISS,
// # of times cache hit when accessing filter block from block cache.
BLOCK_CACHE_FILTER_HIT,
// # of times cache miss when accessing data block from block cache.
BLOCK_CACHE_DATA_MISS,
// # of times cache hit when accessing data block from block cache.
BLOCK_CACHE_DATA_HIT,
// # of bytes read from cache.
BLOCK_CACHE_BYTES_READ,
// # of bytes written into cache.
BLOCK_CACHE_BYTES_WRITE,
```

参考[rocksdb内存使用#block-cache](https://rocksdb.org.cn/doc/Memory-usage-in-RocksDB.html#block-cache)

