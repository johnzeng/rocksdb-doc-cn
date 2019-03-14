RocksDB在一个数据块中查找一个key的时候，使用二分搜索。然而，为了找到数据所在的正确的位置，需要多次key解析和压缩。每个二分搜索导致的CPU缓存未命中，都会导致更高的CPU使用率。在生产环境，我们曾发现这个二分搜索占用了非常可观的CPU使用量。

一个数据块中的哈希索引被设计和开发出来，用于优化点查询中的CPU使用率。使用db_bench的性能测试显示，点查询中的一个主要方法，DataBlockIter::Seek()，的CPU使用率减低了21.8%，在纯缓存压力下，RocksDB的总体吞吐增加了10%，带来的额外开销是4.6%的内存。

# 如何使用

这个功能加入了两个新的选项：BlockBasedTableOptions::data_block_index_type和BlockBasedTableOptions::data_block_hash_table_util_ratio。

哈希索引默认是关闭的，除非设置了BlockBasedTableOptions::data_block_index_type为kDataBlockBinaryAndHash。哈希表的使用率通过BlockBasedTableOptions::data_block_hash_table_util_ratio来配置，同样只有data_block_index_type = kDataBlockBinaryAndHash的时候有效。

```cpp
// the definitions can be found in include/rocksdb/table.h

// The index type that will be used for the data block.
enum DataBlockIndexType : char {
  kDataBlockBinarySearch = 0,  // traditional block type
  kDataBlockBinaryAndHash = 1, // additional hash index
};

DataBlockIndexType data_block_index_type = kDataBlockBinarySearch;

// #entries/#buckets. It is valid only when data_block_hash_index_type is
// kDataBlockBinaryAndHash.
double data_block_hash_table_util_ratio = 0.75;
```

# 需要注意的事情

## 自定义比较器

哈希索引会哈希不同的key（不同的内容，以及字节序列）到不同的哈希值。这就假设了比较器不会把两个内容不同的key认为是相等的。

默认的字节比较器把key按照字典序排列，并且能跟哈希索引友好合作，因为不同的key不会被认为是相等的。然而，有些特别构造的比较器可能会这么做。例如，比如说StringToIntComparator可以把一个字符串转换成整形，然后使用整形来进行比较，key "16" 和 "0x10"在StringToIntComparator看来是相等的，但是他们大概率有两个不同的哈希值。后续的查找其中一种格式的key可能无法找到另一种格式的key。

我们加入了一个新的方法给比较器接口：

```cpp
virtual bool CanKeysWithDifferentByteContentsBeEqual() const { return true; }
```

每个比较器的实现应该覆盖这个函数，并且声明这个比较器应该有的行为。如果一个比较器可能认为不同的key是相等的，这个函数返回true，这样，哈希索引的功能就不会打开了，反之亦此。

注意：为了使用哈希索引功能，你应该 1)有一个比较器，不会认为两个不同的key相等 2)覆盖CanKeysWithDifferentByteContentsBeEqual方法，返回false，这样才能打开哈希索引

## 小util_ratio对数据块缓存的影响

把哈希索引加入到数据块的末尾会消耗数据块缓存的空间，导致实际有效的数据块大小变小，并且增加数据块缓存未命中率。因此，一个非常小util_ratio会导致在一个巨大的数据块缓存未命中，额外的IO会抵消通过哈希缓存索引带来的吞吐增长。另外，当允许压缩（compression），哈希未命中率会增加数据块的解压缩操作，同样也会消耗CPU。因此，如果util_ratio过小，CPU可能反而会增长。最好的util_ratio根据工作压力，数据缓存比率，磁盘贷款，延迟而定。在我们的经验，我们认为util_ratio在0.5 ~ 1之间是比较好的范围，既减小cpu使用，又增加吞吐

# 限制

由于我们用uint8_t来存储二分搜索索引，比如，重启间隔索引，重启间隔的总大小不能大于253（我们保留255和254作为特别标记）。对于有大量重启间隔的块，哈希索引不会被创建，点查询不会使用传统的二分搜索进行。

数据块索引只支持点查询。我们不支持区间查询。区间查询会下推到BinarySeek。

RocksDB支持非常多类型的记录，比如Put,Delete,Merge等等（参考[这里]()了解更多）。目前我们只支持Put和Delete，不支持Merge。我们内部有一个限制的支持记录类型集合：

```
kPutRecord,          <=== 支持
kDeleteRecord,       <=== 支持
kSingleDeleteRecord, <=== 支持
kTypeBlobIndex,      <=== 支持
```

对于不支持的记录，搜索过程会下降到传统的二分搜索。

