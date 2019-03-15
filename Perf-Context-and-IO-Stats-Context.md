性能与IO上下文可以帮助用户理解每个DB操作的性能瓶颈。Options.statistics存储了从打开DB开始的所有线程的所有操作累积下来的统计信息。性能和IO统计上下文则针对每个独立的操作进行分析。

这里是性能上下文的头文件[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_context.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_context.h)

这里是IO信息上下文的头文件：
[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/iostats_context.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/iostats_context.h)

他们的剖析等级受下面这个文件中的同一个方法控制：
[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_level.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_level.h)

性能和IO上下文使用同样的机制。惟一的区别是性能上下文测量RocksDB的函数，而IO状态上下文测量IO相关的调用。这些功能需要在那些等待被剖析的查询被执行的线程中打开。如果剖析等级比disable高，rocksdb会更新一个线程本地数据结构的计数器。查询之后，我们可以从该结构中读取计数器信息。

# 如何使用

这里有一个使用性能和IO上下文的典型的例子：

```
#include “rocksdb/iostat_context.h”
#include “rocksdb/perf_context.h”

rocksdb::SetPerfLevel(rocksdb::PerfLevel::kEnableTimeExceptForMutex);

rocksdb:: get_perf_context()->Reset();
rocksdb::get_iostats_context()->Reset();

... // run your query

rocksdb::SetPerfLevel(rocksdb::PerfLevel::kDisable);

... // evaluate or report variables of rocksdb::get_perf_context() and/or rocksdb::get_iostats_context()
```

注意同样的剖析等级会被应用于性能上下文和IO上下文。

你还可以调用rocksdb::get_perf_context->ToString()和rocksdb::get_iostats_context->ToString()来得到一个易读的报告。

# 剖析等级和开销

就跟平时一样，统计数量和开销之间总有一个权衡关系，我们设计了多个剖析等级供你选择：

- kEnableCount 只打开计数器
- kEnableTimeExceptForMutex 打开计数器统计和大多数时间开销统计，除了那些需要在一个共享互斥锁中调用一个函数的时间。
- kEnableTime 进一步增加互斥锁请求和等待的时间。

kEnableCount避免进一步的昂贵的系统时间获取开销，会带来更小的额外开销。我们通常使用这个等级测量所有的操作，并且在某些计数器不正常的时候报告他们。

使用kEnableTimeExceptForMutex，RocksDB的一次操作可能会调用好几次计时函数。我们通常的实践方式是在需要采样的地方打开，或者当用户需要的时候才打开。用户需要非常小心地选择采样率，因为计时函数在不同的平台的开销差异比较大。

kEnableTime进一步允许了在共享互斥锁里面的计时，但是对一个操作的剖析可能会拖慢其他操作。当我们怀疑互斥锁是性能瓶颈的时候，我们使用这个等级来验证问题。

我们如何处理那些在某个等级上被关闭的计数器？如果一个计数器被关闭了，我们仅仅是不更新他们。

# 统计

我们给出一些典型的例子，讲解怎么使用这些信息来解决你的问题。我们不会介绍所有的统计信息。一个完整的对所有统计信息的描述可以再头文件中找到。

## 性能上下文

### 二分搜索开销

user_key_comparison_count帮助我们找出一个二分搜索里面太多的比较是不是问题的根源，特别是当一个更加昂为的比较器被使用的时候。更进一步，由于比较的次数通常因memtable的大小，level 0的SST文件的大小和其他层的大小 而有所差异，但是一个显著增加的计数器意味着有一个不合预期的LSM树结构。你可能需要检查落盘、压缩是不是能保持写速度。

### 块缓存和OS页缓存效率

block_cache_hit_count告诉我们从块缓存中读取数据块的次数，block_read_count告诉我们我们不得从文件系统中读取块的次数（不管块缓存被关闭了还是缓存未命中）。我们可以通过观察这两个值计算快缓存效率。

block_read_byte告诉我们有多少byte数据是从文件系统上读取的。他可以告诉我们一个慢查询是不是因为大量块需要从文件系统读取导致。索引和bloom过滤块通常是巨大的块。一个巨大的块也可以因为有一个非常大键值对。

在非常多RocksDB的设置中，我们依赖OS的页缓存来减少设备IO。事实上，在多数通用硬件设置上，我们建议用户把OS的页缓存设置到足够大来保存除了最底层以外的所有数据，这样我们可以限制一个读查询在一个IO内完成。在这种设定下，我们会发起多个文件系统调用，但是只有一个会真正读取设备。为了验证是否如此，我们可以使用计数器block_read_time来检查花费在从文件系统读取块的次数是不是如我们预期的一样。

### 墓碑

当删除一个key，RocksDB仅仅是加一个成为墓碑的标记，到memtable。在我们真正把包含有这个墓碑的key的文件压缩之前，原始的key都不会被删除。墓碑可能在原始的值被删除之后都还存在。所以如果我们有大量的连续的key被删除，一个用户可能在迭代通过这些墓碑的时候遇到慢查询。计数器internal_delete_skipped_count告诉我们有多少个墓碑被我们跳过了。internal_key_skipped_count则覆盖其他被我们跳过的key。

### Get步骤

我们可以使用"get_*"统计在一个Get查询中的步骤。最重要的两个是get_from_memtable_time和get_from_output_files_time。计数器告诉我们这个慢查询是不是因为memtable，SST文件，或者两者都有的原因导致。seek_on_memtable_time可以告诉我们花费在搜索memtable的时间。

### 写步骤

"write_*"统计写操作过程中的写步骤。write_wal_time，write_memtable_time和write_delay_time告诉我们花费在写WAL，memtable的时间，或者出在减速激活状态。write_pre_and_post_process_time主要意味着花费在写队列的等待时间。如果写操作被赋值给一个提交组，但是他不是组长，write_pre_and_post_process_time会包括等待组提交组长的时间。

### 迭代操作步骤

"seek_*" 和 find_next_user_entry_time把迭代操作分步。最有趣的一个是seek_child_seek_count，他告诉我们有多少子迭代，通常也就是LSM树的排序结果数量。

## IO统计上下文

我们有计数器来统计在主要文件系统调用的时间花费。如果你发现写路径有问题，写相关的计数器通常更有趣。他告诉我们，我们因为哪个文件系统调用而变慢，或者他不是因为文件系统调用导致的

