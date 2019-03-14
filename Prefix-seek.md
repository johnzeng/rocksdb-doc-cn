# 前缀搜索接口

如果你的DB或者列族的options.prefix_extractor选项有被声明，那么rocksdb就会在一个“前缀搜索”模式，具体会在下面解释。使用的例子如下：

```cpp
Options options;

// <---- Enable some features supporting prefix extraction
options.prefix_extractor.reset(NewFixedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"
iter->Next(); // Find next key-value pair inside prefix "foo"

```
options.prefix_extractor是一个SliceTransform类型的共享指针。通过调用SliceTransform.Transform()，我们可以我们从一个Slice里面提取出一个子串，用来代表该Slice，通常是前缀部分。在这个wiki页面，我们使用“前缀”来指代 options.prefix_extractor.Transform() 对一个 key 处理后的输出。你可以通过调用NewFixedPrefixTransform(prefix_len)，获得一个使用固定长度的前缀转换器，或者你可以根据需要实现自己的转换器，然后传给options.prefix_extractor。

当options.prefix_extractor.不是nullptr，迭代器不能保证所有key都是按顺序迭代的，只能保证相同前缀的key是按顺序迭代的。当调用Iterator.Seek(lookup_key)时，RocksDB会从lookup_key里面提取前缀。与全排序模式不同，如果数据库里面有一个或者多个key满足这个前缀，RocksDB会把迭代器放在前缀相同，或者更大的key上。如果没有前缀等于或者大于lookup_key的前缀，或者在调用几次Next之后，根据ReadOptions.prefix_same_as_start是否为true，我们处理完了相同前缀的所有key，我们可能会反回Valid()=false，或者是任何一个大于前面的key的key。从4.11之后，我们支持前缀模式下使用Prev，但是仅限于迭代器仍旧在该前缀的所有键值范围内的时候。Prev的输出在迭代器已经超出范围的时候不保证正确。

当前缀模式被打开的时候，RocksDB会为了快速定位相同前缀的key或者排除不存在的前缀，自由组织数据，或者构建搜索数据。这里有些为前缀模式进行优化的支持：数据块和memtable的prefix bloom，基于哈希的memtable，还有平表格式。一个示范设定：

```cpp
Options options;

// Enable prefix bloom for mem tables
options.prefix_extractor.reset(NewFixedPrefixTransform(3));
options.memtable_prefix_bloom_bits = 100000000;
options.memtable_prefix_bloom_probes = 6;

// Enable prefix bloom for SST files
BlockBasedTableOptions table_options;
table_options.filter_policy.reset(NewBloomFilterPolicy(10, true));
options.table_factory.reset(NewBlockBasedTableFactory(table_options));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"

```
从3.5版本开始，我们支持通过一个配置，使得rocksdb即使在前缀模式，也能使用全顺序。调用NewIterator的时候，打开ReadOption.total_order_seek=true就可以打开这个功能。

```cpp
ReadOptions read_options;
read_options.total_order_seek = true;
auto iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```
这个模式下，性能可能会变差。请注意，并不是所有的前缀搜索实现都支持这个功能。例如，平表的实现就不支持，所以如果你尝试这么使用，你会看到一个错误码返回。基于哈希的mentable会在使用这个功能的时候做一次昂贵的在线排序。prefix bloom和使用哈希索引的基于块的表是支持这个模式的。

## 限制

SeekToLast对前缀索引不友好。SeekToFirst只对部分配置有支持。如果你的迭代器需要使用这两个操作，你应该使用全排序模式。

一个常见的使用前缀索引的bug是使用逆序迭代前缀模式。这个还没有支持。如果你需要经常使用逆序迭代，你可以重新对数据进行排序，然后把前缀迭代器的顺序反过来。你可以通过自己实现一个比较器，或者使用其他方法编码你的key

## API变化（2.8->3.0）

这一节，我们会说明从2.8版本到3.0版本的API变化

### 修改前

#### 全排序搜索

这就是你认为的传统的索引行为。搜索会在一个全排序的key空间进行，把迭代器定位到第一个大于或者等于你搜索的key的位置。

```cpp
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);

```
并不是所有表组织方式都支持全排序搜索。例如，新引入的平表格式，除非使用全排序模式打开（Options.prefix_extractor == nullptr），否则就只支持基于前缀的seek。

#### 使用ReadOptions.prefix

这是最不灵活的搜索方式。创建迭代器的时候需要支持前缀。

```cpp
Slice prefix = "foo";
ReadOptions ro;
ro.prefix = &prefix;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar"
iter->Seek(key);
```

Options.prefix_extractor需要提前设置。Seek调用会被ReadOptions提供的前缀限制，这意味着，如果你希望搜索一个新的前缀，你需要重新创建迭代器。这种方式的好处是，不相关的文件可以在创建迭代器的时候被过滤掉。所以如果你希望搜索同一个前缀的不同key，他的表现会比较好。然而，我们认为这个使用方式非常少见。

#### 使用ReadOptions.prefix_seek

这个模式比 ReadOption.prefix 更加灵活。创建迭代器的时候没有预过滤。这样，通过一个迭代器就可以用于搜索不同的key和前缀了。

```cpp
ReadOptions ro;
ro.prefix_seek = true;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar";
iter->Seek(key);
```

与ReadOptions.prefix一样，Options.prefix_extractor是前置条件。

### 修改的内容

很显然，三种搜索模式让人感到混乱：

- 一个模式要求另一个选项被设置（例如Options.prefix_extractor）
- 对于我们的用户而言，后面两种模式哪一个在不同的场景下比较合适，不明显。

这个修改希望解决这个问题，并且把事情变得简单明了：如果Options.prefix_extractor被设置，seek默认使用前缀模式，反之亦然。动机很简单：如果Options.prefix_extractor存在，很显然，数据可以被切片，然后前缀搜索就自然匹配这种场景。使用方式就变的统一了：

```cpp
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);

```

### 迁移到新的用法

迁移到新的用法很简单。去除Options.prefix或者Options.prefix_seek的赋值，因为他们已经被弃用了。现在，直接搜索你的key或者前缀就好了。因为Next会穿过上限，走到不同的前缀去，你可能需要检查结束状态：

```cpp
    auto iter = DB::NewIterator(ReadOptions());
    for (iter.Seek(prefix); iter.Valid() && iter.key().starts_with(prefix); iter.Next()) {
       // do something
    }
```

