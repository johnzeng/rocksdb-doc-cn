模拟缓存（SimCache）可以帮助用户预测块缓存 在当前工作压力下，在一个特定的模拟（内存）大小下，的性能，比如，命中率，未命中率，而不用实际使用特定数量的内存。

# 动机

可以帮助用户调优块缓存大小，以及确定他们使用块缓存的效率。同事，帮助理解在高速存储下的缓存性能。

# 介绍

SimCache的基本思想是，把普通的块缓存封装成一个使用目标模拟大小的，只有key的块缓存。当插入的时候，我们把key查到cache，但是值只会插入到普通缓存。这样，值得大小就同事对两个缓存有影响，这样我们模拟了一个特定大小的块缓存的行为，而不需要使用实际的内存，实际内存使用只涉及key的总大小。

# 如何使用SimCache

由于SimCache是一个普通块缓存的封装。用户需要使用[NewLRUCache](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/cache.h)创建一个块缓存:

```
std::shared_ptr<rocksdb::Cache> normal_block_cache =
  NewLRUCache(1024 * 1024 * 1024 /* capacity 1GB */);
```

然后使用[NewSimCache](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/utilities/sim_cache.h)封装normal_block_cache，然后把SimCache设置为rocksdb::BlockBasedTableOptions的block_cache字段，并且生成options.table_factory：

```
rocksdb::Options options;
rocksdb::BlockBasedTableOptions bbt_opts;
std::shared_ptr<rocksdb::Cache> sim_cache = 
NewSimCache(normal_block_cache, 
  10 * 1024 * 1024 * 1024 /* sim_capacity 10GB */);
bbt_opts.block_cache = sim_cache;
options.table_factory.reset(new BlockBasedTableFactory(bbt_opts));
```

最后，使用该选项打开DB。然后SimCache的HIT/MISS值可以分别通过sim_cache->get_hit_counter()和sim_cache->get_miss_counter()获得。可选的，如果你不希望存储sim_cache，并且你的Rocksdb版本大于v4.12，你可以通过[rocksdb::Statistic](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/statistics.h)滴答计数器SIM_BLOCK_CACHE_HIT和SIM_BLOCK_CACHE_MISS获取这些统计信息。

# 内存使用

人们可能会担心SimCache实际的内存使用，大概包括：

sim_capacity * entry_size / (entry_size + block_size),

- 76 <= entry_size (key_size + other) <= 104
- BlockBasedTableOptions.block_size = 4096。默认值，可配置

因此，默认SimCache的内存使用为**sim_capacity * 2%**。


