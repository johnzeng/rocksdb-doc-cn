通过DB::CompactRange 或者 DB::CompactFiles，可以手动触发压缩。这是给高级用户开发自定义压缩策略使用的，包括但不限于以下使用方法：

- 通过在读取大量数据之后，把文件压缩到最底层，来优化读多写少的工作压力。
- 强制数据进入压缩过滤器，以固定数据。
- 迁移到一个新的压缩配置。例如，如果修改了层数量，可以调用`CompactRange`来把数据压缩到最底层，然后把文件移动到目标层。

下面的例子展现如何使用这些API。

```cpp
Options dbOptions;

DB* db;
Status s = DB::Open(dbOptions, "/tmp/rocksdb",  &db);

// Write some data
...
Slice begin("key1");
Slice end("key100");
CompactRangeOptions options;

s = db->CompactRange(options, &begin, &end);
```

或者

```cpp
CompactionOptions options;
std::vector<std::string> input_file_names;
int output_level;
...
Status s = db->CompactFiles(options, input_file_names, output_level);

```


# CompactRange

begin和end参数定义需要压缩的key的范围。根据db使用的压缩风格而有不同的行为。在universal和FIFO压缩风格，begin和end参数会被忽略，所有文件都会被压缩。另外，每一层的文件都会被压缩，并且留在本层。对于leveled压缩风格，所有包含有key范围的文件都会被压缩到最底层。如果begin或者end为NULL，这意味着使用第一个的key或者最后一个的key。

如果多于一个线程调用了人工压缩，只有一个会真正被调度，而其他线程会等待已经调度的压缩完成。如果CompactRangeOptions::exclusive_manual_compaction被设置为true，调用会禁止自动压缩工作的调度，然后等待已经开始的自动压缩工作停止。

CompactRangeOptions支持以下选项：

- CompactRangeOptions::exclusive_manual_compaction。 为true时，如果人工压缩开始了，就不会有其他压缩进行了。默认为true
- CompactRangeOptions::change_level,CompactRangeOptions::target_level。 两个选项一起决定压缩后的文件会放在那一层。如果target_level为-1，压缩文件会移动到层号最小的，max_bytes能满足放下所有文件的层。中间的层必须为空。例如，如果文件初始化被压缩到L5，然后L2是最小的、能存放所有数据的层，那么如果L3和L4是空的，他们会被放在L2，或者如果L3不是空的，就放在L4。如果target_level为正，压缩后的文件会放在那个层，并且中间层为空。如果中间的层没有空的，那么压缩的文件会被放在他们原来的层。
- CompactRangeOptions::target_path_id 压缩输出会被放在options.db_paths[target_path_id]目录。
- CompactRangeOptions::bottommost_level_compaction，当设置为BottommostLevelCompaction::kSkip，或者设置为 BottommostLevelCompaction::kIfHaveCompactionFilter ，并且这个列族有压缩过滤器被定义，那么最底层的文件不会被压缩

## CompactFiles

这个接口会压缩所有输入文件到一系列输出文件，然后放在output_level中。输出文件的大小取决于数据的大小以及CompactionOptions::output_file_size_limit的设定。这个API在ROCKSDB_LITE里面不支持。

