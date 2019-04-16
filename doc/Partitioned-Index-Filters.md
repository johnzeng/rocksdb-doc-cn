随着DB/内存比例变大，过滤器/索引块的内存空间变得越来越重要。尽管cache_index_and_filter_blocks允许在块缓存中只存储这些内容的一个子集，但是他们本身巨大的尺寸会通过以下方式降低性能：

- 占用本来是用于缓存数据的块缓存空间。
- 在未命中的时候，加载数据到内存，增加磁盘存储的压力。

这里我们讲解释问题的细节，以及如何给索引和过滤器分片，以减少消耗。

## 索引/过滤器块有多大？

RocksDB默认对每个SST文件有一个索引/过滤器块。根据配置的不同，索引/过滤器块的大小也会有所不同，但是，对于一个256MB的SST文件，通常他的索引/过滤器块的大小就是0.5/5MB，比常见的4~32KB的数据块要大很多。如果索引/过滤器块能完美的加载进内存，那是完全可以的，这样他们在SST的生命周期中只要被读取一次就可以，而不是需要跟数据块竞争块缓存空间，导致他们需要被重复从磁盘读取。

## 巨大的索引/过滤器块会有什么大问题吗？

如果索引/过滤器块存储在块缓存，他们会跟数据块（以及他们自己）竞争这部分稀有资源。一个5MB的过滤器占用的空间可以被1000多个数据块（4KB大小）使用。这会导致数据块出现更多的缓存未命中。巨大的索引/过滤器同时还会把对方踢出块缓存，导致他们自己的缓存命中率下降。这会导致只有一小部分的索引/过滤器块会在他的缓存生命周期内被实际使用。

在一个索引/过滤器的缓存未命中发生后，他需要从磁盘中重新加载，他巨大的尺寸不能帮助减轻IO开销。就算是一个简单的点查询也需要从LSM的每一层读取好几个数据块，这也可能导致载入大量的索引/过滤器块。如果这种情况经常发生，磁盘甚至可能会花费比 服务数据块 更多的时间来服务索引/过滤器。

## 什么是分片索引/过滤器？

使用分片，一个SST文件的索引/过滤器会被分片成多个更小的块，然后加入一个新的上层索引给他们。当读取一个索引/过滤器的时候，只有最上层的索引被加载到内存。分片索引/过滤器之后会使用顶层索引来按需加载该查询需要使用的索引/过滤器到块缓存。顶层索引有更小的内存空间，可以被存储在堆上，或者是块缓存，具体配置依赖于cache_index_and_filter_blocks。

### 优点:

- 更高的缓存命中率：不再使用巨型的索引/过滤器块污染缓存，分片允许索引/过滤器以一个更小的单位加载，因此加大了缓存空间利用率。
- 更小的IO单位：当一个分片的索引/过滤器块缓存未命中发生的时候，只有一个分片需要从磁盘被载入内存，相比较读取整个索引/过滤器，他大大减小了磁盘压力。
- 不需要牺牲索引/过滤器：不使用分片直接降低索引/过滤器内存使用的的策略通常是牺牲他们的准确性，比如说，使用更大的数据块或者更少的额bloom位来生成一个更小的索引和过滤器。

### 缺点:

- 存储顶级索引的额外磁盘空间： 挺小的，大概是0.1~1%的索引/过滤器大小。
- 更多的磁盘IO：如果顶级索引不在缓存，他会需要一个额外的IO。为了避免这种情况，他们可以存放在堆里，或者是高优先级缓存（TODO的工作）。
- 失去空间局部性：如果有一个工作压力，频繁的请求，然后从同一个SST文件做随机读取，会导致每次读取都要额外读取一次独立的索引/过滤器，这比直接把整个索引读取起来要低效一些。不过我们在我们的压力测试中没有看到这种情况，他通常发生在LSM树的L0/L1层，这里分片可以关闭（TODO）

## 成功案例

### HDD, 100TB DB

在这个例子里，我们有一个86G的DB存放在HDD上，用这个DB，使用直接IO，来模拟一个100TB数据的节点的小内存（跳过OS的文件缓存），使用块大小60MB。分片把性能从 5 op/s 提升了大概11倍吞吐到55ops/s。

```
./db_bench --benchmarks="readwhilewriting[X3],stats" --use_direct_reads=1 -compaction_readahead_size 1048576 --use_existing_db --num=2000000000 --duration 600 --cache_size=62914560 -cache_index_and_filter_blocks=false -statistics -histogram -bloom_bits=10 -target_file_size_base=268435456 -block_size=32768 -threads 32 -partition_filters -partition_indexes -index_per_partition 100 -pin_l0_filter_and_index_blocks_in_cache -benchmark_write_rate_limit 204800 -max_bytes_for_level_base 134217728 -cache_high_pri_pool_ratio 0.9
```


### SSD,Linkbench

在这个例子里我们有一个SSD上的300GB的数据库，同样通过直接IO（跳过OS文件缓存）扮演本地另一个DB的小内存，块缓存大小为6G和2G。不使用分片的时候，把块缓存大小从6G调到2G，linkbench的吞吐从38k tps掉到 23kb。使用分片，吞吐下降为从38k掉到30k。

## 如何使用？

- index_type = IndexType::kTwoLevelIndexSearch
	- 打开分片索引
- NewBloomFilterPolicy(BITS, false)
	- 使用全过滤器
- partition_filters = true
	- 为了打开分片过滤器
- metadata_block_size = 4096
	- 索引分片的块大小
- cache_index_and_filter_blocks = false [如果你的版本 <= 5.14]
	- 分片总是存在块缓存中。这是为了控制顶级索引的位置（可以更好的填入内存）：固定在堆还是缓存到块缓存中。存储到块缓存比较少用。
- cache_index_and_filter_blocks = true and pin_top_level_index_and_filter = true [如果你的版本 >= 5.15]
	- 把所有东西都丢进块缓存，同时固定顶级索引，这个索引不会很大的。
- cache_index_and_filter_blocks_with_high_priority = true
	- 推荐配置
- pin_l0_filter_and_index_blocks_in_cache = true
	- 推荐设置，因为这个属性可以拓展到索引/过滤器分片。
	- 只有压缩方式是基于level的压缩方式的时候才使用他
	-  **注意**:如果固定这些块到块缓存，如果strict_capacity_limit没有设置（默认），可能会使容量过大。
-  块缓存大小：如果你习惯于把过滤器/索引存储在堆里，不要忘了把你从堆那边节省出来的缓存归还给块缓存。

## 当前限制

- 分片过滤器如果不使用分片索引，就不能打开。
- 我们有一样数量的过滤器和索引分片。换句话说，不管索引块怎么切，过滤器块也那么切。如果这样有问题，我们以后可能会考虑改掉他。
- 过滤器块大小是根据索引块切分的时候决定的。我们会尽快拓展metadata_block_size来限制过滤器和索引块的最大大小，比如，一个过滤器块会在 索引块被切割，或者，他的大小可能超过metadata_block_size的时候，被切割

# 底层实现

这里展示开发者关心的实现细节。

## BlockBasedTable格式

可以在[这里]()了解BlockBasedTable格式。使用分片的时候，差异会是索引块[index block]，会被存储成：

```
[index block - partition 1]
[index block - partition 2]
...
[index block - partition N]
[index block - top-level index]
```

然后SST的脚注指向顶级索引块（他自身就是一个索引分片块的索引）。每个独立的索引块分片遵从与kBinarySearch相同的格式。顶级索引格式同事遵从kBinarySearch，因此可以使用普通的数据块读取器来读取。

类似的结果被用于过滤器块分片。每个独立的过滤器块的格式遵从kFullFilter的格式。顶级索引格式遵从kBinarySearch的格式，类似索引块的顶级索引。

注意，对于带分片的SST文件，检查SST的工具，如sst_dump，会报告索引/过滤器上的顶级索引的大小，而不是索引和过滤器块的总大小。


## 构造器

分片索引和过滤器可以通过PartitionedIndexBuilder和PartitionedFilterBlockBuilder分别构造。

PartitionedIndexBuilder维护sub_index_builder_，一个指向ShortenedIndexBuilder的指针，用来构建当前的索引分片。当通过flush_policy_指定的时候，构造器会把指针跟索引块中最后一个key一起保存，然后创建一个新的活跃ShortenedIndexBuilder。当你对这个构造器调用::Finish，他会对最早的子索引构造器调用::Finish然后返回分片块的结果。下次调用PartitionedIndexBuilder::Finish同样会包含之前返回的SST文件分片的偏移，会被用作顶级索引的值。最后一次PartitionedIndexBuilder::Finish 的调用会完成顶级索引，这次会返回该顶级索引。把顶级索引存储到SST文件之后，他的偏移会被用作索引块的偏移。

PartitionedFilterBlockBuilder继承FullFilterBlockBuilder，它带有一个FilterBitsBuilder可以用于构建bloom过滤器。他也有一个指针指向PartitionedIndexBuilder，并且使用该指针调用ShouldCutFilterBlock，来决定什么时候应该吧一个过滤器块切片（在一个索引块被切片后）。为了切割一个过滤器块，他执行完FilterBitsBuilder并且把 返回的块 和 PartitionedIndexBuilder::GetPartitionKey()提供的分片key 存储在一起，然后重置FilterBitsBuilder，一边下一个分片使用。在最后一次 PartitionedFilterBlockBuilder::Finish 被调用的时候，其中一个分片会返回，并且前面分片的偏移会被用于构建顶级索引。调用::Finish会返回顶级索引块。

之所以基于PartitionedIndexBuilder实现PartitionedFilterBlockBuilder，是为了优化索引/过滤器分片在SST文件的插入。不过这个优化效果不是很明显，我们以后可能会放弃这个依赖。

## 读取器

分片索引通过PartitionIndexReader来读取，他会操作顶级索引块。当NewIterator被调用，一个TwoLevelIterator会作用于顶级索引块。这个简单的实现的原理是这样的，因为每个索引分片都是kBinarySearch格式，他们跟数据块是一样的，因此他们可以简单地插入一层迭代器来实现迭代。如果pin_l0_filter_and_index_blocks_in_cache被设置了，底层的迭代器会被固定到PartitionIndexReader，这样，只要PartitionIndexReader还存活，他们对应的索引分片都会被固定到块缓存。BlockEntryIteratorState使用一系列的固定的分片偏移来避免两次解除一个索引分片的固定。

PartitionedFilterBlockReader使用顶级索引找到过滤器分片的偏移。然后调用BlockBasedTable对象的GetFilter，通过FilterBlockReader对象读取块缓存上的过滤器分片（如果没有，则从磁盘读入到这里来） ，然后释放FilterBlockReader对象。为了让table_options.pin_l0_filter_and_index_blocks_in_cache支持分片索引，PartitionedFilterBlockReader并不会释放这些块的缓存句柄（或者说，保证他们会被固定到块缓存）。他从另一个方向维护了filter_cache_，一个固定的FilterBlockReader映射表，被用于在PartitionedFilterBlockReader析构的时候释放缓存项。

