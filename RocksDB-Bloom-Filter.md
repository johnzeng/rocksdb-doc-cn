# 什么是Bloom过滤器

对于任意集合的key，一个可以用来构建一个bit数组的算法成为Bloom过滤器。给出任意key，这个bit数组可以用于决定这个key是不是*可能存在*或者*绝对不存在*于这个key集合。对于更多的细节介绍Bloom过滤器如何工作，参考这个[维基百科](http://en.wikipedia.org/wiki/Bloom_filter)

在RocksDB，当过滤策略被设置，每个新建立的SST文件会带有一个Bloom过滤器，被用于决定文件是否会包含我们在查找的key。过滤器本质上是一个bit数组。多个哈希函数会被作用在这些key上，每个决定一个bit数组中的一个是否会被set为1。读取的时候，同样的哈希函数会被应用在每个key，bit数组都会被检查，探测，然后如果有一个探测返回0，那么这个key肯定不在这个集合里。

# 生命周期

在RocksDB，每个SST文件有对应的Bloom过滤器。这个过滤器在SST文件写入存储的时候被创建，并且作为SST文件的一部分存储。Bloom过滤器在每一个层都以相同的方式构建（注意，通过设置optimize_filters_for_hits，最后一层的bloom可能会被选择性跳过）

Bloom过滤器可能只会通过一个key集合被创建——没有操作能结合不同的Bloom过滤器。当我们合并两个SST文件，一个新的Bloom过滤器会根据新的文件里的key进行创建。

当我们打开一个sst文件的时候，对应的bloom过滤器会同样被打开并且加载到内存。当SST文件被关闭，bloom过滤器会从内存移除。如果希望把Bloom过滤器缓存到块缓冲区，使用BlockBasedTableOptions::cache_index_and_filter_blocks=true

# 基于块的bloom过滤器（旧格式）

一个Bloom过滤器只有在所有的key都能放入内存的时候被构建。换句话说，把一个bloom过滤器分片并不影响他的假阳性概率。因此，为了减轻构建SST文件的时候的内存压力，在旧的格式中，每2KB的键值对块，就建立一个独立的bloom过滤器。这个格式的细节参考[这里]()。最后，一个存储有每个bloom块的偏移的数组会被存储在SST文件中。

读取的时候，可能含有kv对的块的偏移会从SST索引中获取。根据偏移，对应的bloom过滤器会被加载。如果过滤器的结果显示key可能存在，那么他才搜索那个key实际的数据块。

# 全过滤（新格式）

旧的格式的每个过滤块没有根据缓存进行对齐，搜索的时候可能会导致大量缓存未命中。更重要的是，尽管过滤器的阴性响应（也就是说key不存在）节省了数据块搜索的时间，但是索引还是会被加载到内存。新的格式，全过滤，通过构建一个针对整个SST文件的过滤器来解决这个问题。代价就是需要更多的缓存空间来存储文件中的每个key的哈希（在旧格式，只需要缓存2KB的块的key）

全过滤限制一个key的探测bit到一个相同CPU缓存流水线中。通过限制每个key的CPU缓存未命中率，保证了快速搜索。注意这本质上是把bloom空间分片并且只要我们有足够多的key，就不会影响假阳性概率。参考“数学计算”章节了解更多细节。

读取的时候，RocksDB使用与构建SST文件的时候同样的格式。用户可以设置filter_policy选项，声明新创建的SST文件的格式。helper函数NewBloomFilterPolicy可以用于帮助构建新老的基于块的过滤器（默认）以及新的全过滤器。

```cpp
extern const FilterPolicy* NewBloomFilterPolicy(int bits_per_key,
    bool use_block_based_builder = true);
}
```

# 前缀 vs 整个key

默认我们会把每个完整的key的哈希结果加入到bloom过滤器。这个可以通过设置BlockBasedTableOptions::whole_key_filtering为false来关闭。当你设置了Options.prefix_extractor，一个前缀的哈希会被加入到bloom中。由于前缀的唯一性比完整的key小，只存储前缀会生成比较小的bloom，不好的一点就是会导致更高的假阳性概率。更重要的是，在使用Seek和SeekForPrefix的时候，前缀bloom也可以用（通过在构建迭代器的时候设置check_filter），而完整key的bloom只能被用于点查询。

# 统计

这里是一些统计信息，来告诉你如何获取你的全bloom过滤在生产环境的表现如何。

如果允许了前缀过滤，并且check_filter被设置，那么在每次::Seek和::SeekForPrev被调用之后会更新下面这些统计条目

-	rocksdb.bloom.filter.prefix.checked: seek_negatives + seek_positives
-	rocksdb.bloom.filter.prefix.useful: seek_negatives

在每次点查询后，会更新下面的条目。如果whole_key_filtering被设置，这是检查完整key的bloom的结果，否则这是检查前缀bloom的结果：

-	rocksdb.bloom.filter.useful: [true] negatives
-	rocksdb.bloom.filter.full.positive: positives
-	rocksdb.bloom.filter.full.true.positive: true positives

点查询的假阳性率可以通过(positives - true positives) / (positives - true positives + true negatives)来计算。

注意这些让人迷惑的场景：

- 如果whole_key_filtering和prefix被设置，prefix在点查询的时候不会被检查
- 如果只有prefix被设置，前缀bloom被校验的总次数为点查询和seeks的和。由于seek中真阳性结果的缺失，我们无法得到总的假阳性率：只能拿到点查询的。

# 自定义FilterPolicy

FilterPolicy(include/rocksdb/filter_policy.h)可以被继承，用于定义自己的过滤器。有两个主要的方法需要实现：

```
FilterBitsBuilder* GetFilterBitsBuilder()
FilterBitsReader* GetFilterBitsReader(const Slice& contents)
```

这样，新的过滤策略会以一个FilterBitsBuilder和FilterBitsReader的工厂来运作。FilterBitsBuilder提供key存储与过滤器生成的接口，FilterBitsReader提供校验key是否存在于过滤器中的接口。

注意：这两个新的接口只能在新的过滤器格式中工作。旧的过滤器仍旧使用旧的方法来自定义。

# 分片bloom过滤器

这一全过滤的一种存储格式。他把过滤器块分片成记个小的块来缓解块缓存的加载。参考[这里]()

# 数学

这里是关于bloom过滤器的假阴性率（FPR）的数学计算部分

- m bit且分到s个分片，每个大小为m/s
- n个key
- k 个 探测/key
- 对于每个key，随机选择一个分片，k bit会在分片中随机设置
- 插入一个key之后，某个特定的bit仍旧为0的可能性 是 所有之前的key的哈希要么有 (s-1)/s的可能性在其他分片被设置，要么 该分片有1-1/(m/s)的概率设置其他bit
- 使用二分逼近法 (1+x)^k ~= 1+ xk if |xk| << 1并且|x| < 1，我们有
	- prob_0 = 1-1/s + 1/s (1-sk/m) = 1 - k/m = (1-1/m)^k
- 注意到prob_0(s=1)的近似值等于prob_0
- 插入n个key之后，一个bit仍旧是0的概率是：
	 - prob_0_n = (1-1/m)^kn
- 这样假阳性的概率就是所有的k个bit为1：
	- FPR = (1 - prob_0_n) ^ k = (1- (1-1/m)^kn) ^ k

注意，FPR率不依赖于s，分片的数量。换句话说，只要sk/m << 1，分片不影响FPR。对于典型的数值s=512 且 k=6, 然后 每个key 10 bit，只要n >> 307，就能满足条件。在全过滤中，每个分片受CPU缓冲流水线允许的对齐大小和减去下次探测的cpu缓存不命中的影响。m会被设置为n * bits_per_key + epsilon来保证他是一个分片大小的整数倍，也就是cpu缓冲流水线的大小。


PS.译者注：最后一段最后一句确实没看懂该怎么翻译。。。有知情人士请联系

