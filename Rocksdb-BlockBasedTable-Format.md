本页面从LevelDB的文档[表格式](https://github.com/google/leveldb/blob/master/doc/table_format.md) fork出来，然后修改了我们在开发rocksdb的时候修改的部分。

RocksDB中默认的SST表格式是BlockBasedTable。

# 文件格式

```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter块]        (参考章节: "filter" Meta Block)
[meta block 2: stats块]         (参考章节: "properties" Meta Block)
[meta block 3: 压缩字典块]       (参考章节: "compression dictionary" Meta Block)
[meta block 4: 范围删除块]       (参考章节: "range deletion" Meta Block)
...
[meta block K: 未来拓展块]       (我们以后可能会加入新的元数据块)
[metaindex block]
[index block]
[Footer]                               (定长脚注，从file_size - sizeof(Footer)开始)
<end_of_file>
```

文件包含内部指针，调用BlockHandles，包含下面信息：

```
offset:         varint64
size:           varint64
```

参考这个 [文件](https://developers.google.com/protocol-buffers/docs/encoding#varints)了解varint64格式

(1) 键值对序列在文件中是以排序顺序排列的，并且切分成了一序列的数据块。这些块在文件头一个接一个地排列。每个数据块都根据block_builder.cc的代码进行编排（参考文件中的注释），然后选择性压缩（compress）

(2) 数据块之后，我们存出一系列的元数据块。支持的元数据块类型在下面描述。更多的数据块类型可能会在以后加入。同样的，每个元数据块都根据block_builder.cc的代码进行编排（参考文件中的注释），然后选择性压缩（compress）

(3) 一个metaindex块对每个元数据块都有一个对应的入口项，key为meta块的名字，值是一个BlockHandle
，指向具体的元数据块。

(4) 一个索引块，对每个数据块有一个对应的入口项，key是一个string，该string >= 该数据块的最后一个key，并且小于下一个数据块的第一个key。值是数据块对应的BlockHandle。如果索引类型(IndexType)是[kTwoLevelIndexSearch](https://rocksdb.org.cn/doc/Partitioned-Index-Filters.html)，这个索引块就是索引分片的第二层索引，例如，每个入口指向另一个索引块，该索引块包含每个数据块的索引。在这种情况下，格式就变成了：

```
[index block - 1st level]
[index block - 1st level]
...
[index block - 1st level]
[index block - 2nd level]
```

(5) 在文件的最后的最后，有一个定长的脚注，包含metaindex以及索引块的BlockHandle，同事还有一个魔数：

```
   metaindex_handle: char[p];      // metaindex的Block handle
   index_handle:     char[q];      // 索引块的Block handle
   padding:          char[40-p-q]; // 填充0以达到固定大小
                                   // (40==2*BlockHandle::kMaxEncodedLength)
   magic:            fixed64;      // 0x88e241b785f4cff7 (小端)
```

# 过滤器元数据块

## 全过滤器

在这种filter中，每个SST文件只有一个过滤块。

## 分片过滤器

全过滤器被分片到多个块。一个顶级索引块被加入到映射键表中，用于定位过滤分片。更多信息参考[这里]()

## 基于块的过滤器

基于块的过滤器，被废弃，so，我也不翻译了。。。

# 属性元数据块

这个元数据块包含一系列的属性。key为属性名。值为具体的值。

统计块按下面格式排列：

```
 [prop1]    (每个属性都是一个键值对)
 [prop2]
 ...
 [propN]
```

属性保证排序好，并且没有重复。

默认情况下，每个表提供以下属性：

```
 data size               // 所有数据块的总大小 
 index size              // 索引块的大小
 filter size             // 过滤块的大小.
 raw key size            // 未处理过的key的总大小
 raw value size          // 未处理过的值的总大小
 number of entries
 number of data blocks
```

Rocksdb还提供用户一个“回调”来收集他们对这个表感兴趣的属性。参考UserDefinedPropertiesCollector。

# 压缩字典元数据块

这个元数据块包含一个字典，在压缩库的压缩和解压每个块前导入。他的目标是解决一个基本问题，即小数据块的动态字典压缩算法：字典在块之间一次性构建，所以小的块总是小的，导致字典没啥用。

我们的解决方案是使用一个字典初始化压缩库，字典通过之前看到的数据块的数据样本建立。这个字典之后被排序到一个文件级别的元数据块，以备解压使用。这个字典的大小上限通过CompressionOptions::max_dict_bytes定义。默认为0，也就是这个块不会被生成或者排序。当前这个功能支持kZlibCompression, kLZ4Compression, kLZ4HCCompression, 和kZSTDNotFinalCompression。

更进一步，这个压缩字典只在最底层的压缩（compaction）的时候才生成，这里的数据量最大，同时也最稳定。为了避免多次遍历数据，这个字典只根据自压缩流程的第一个输出文件来取样。然后字典就被应用于压缩，并且作为输出排序好放入元数据块。如果文件比较小，一些样本间距会到超过EOF，也就是这个字典会比CompressionOptions::max_dict_bytes小一些。

# 区间删除元数据块

这个元数据块包含这个文件的键范围和seqnum范围内的区间删除操作。区间删除不可以嵌入到数据块，因为这样做的话就不能使用二分搜索了。

这个快格式是标准的kv格式。一个区间删除以下面方式编码：

-	User key: 区间开始key
-	Sequence number: 区间删除操作被插入到数据库的时候的序列号
-	Value type: kTypeRangeDeletion
-	Value: 区间结束key

区间删除在插入的时候会被赋予一个seqnum，这个seqnum跟其他非区间操作类型（puts,deletes，等）用同样的机制生成。他们还会使用同样的机制在落盘和压缩（compaction）的时候在LSM树中移动。他们只有在被压缩到最后一层的时候才会被淘汰。


