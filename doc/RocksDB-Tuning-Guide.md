本指南的目的是提供你足够的信息用于根据自己的工作负载和系统配置调优RocksDB。

RocksDB非常灵活，这有好也有坏。你可以真多很多工作场景和存储技术进行调优。在Facebook，我们使用相同的代码跑内存工作压力，闪盘设备和机械硬盘。然而，灵活性不总是对用户友好的。我们引入了大量的调优参数，让人疑惑不解。我们希望这个指南会帮助你压榨你的系统的最后一滴性能并且完全利用你的资源。

我们假设你有一定的基础知识，了解LSM工作原理。关于LSM的资源非常多，不需要再写一个了。

# 放大因子

调优RocksDB通常就是在三个放大因子间做权衡：写放大，读放大，和空间放大。

**写放大**是 写入磁盘的数据 与 写入数据库的字节数的比。

例如，如果你写入 10MS/s 到数据库，然后你观察到硬盘写速度为30MB/s，你的写放大为3.如果写放大很高，工作负载的瓶颈可能在磁盘吞吐。比如，如果写放大是50，而磁盘吞吐是500MB/s，你的数据库只能达到10MB/s的写速度。在这种情况下，减少写放大会直接增加最大写速率。

高写放大同时减少闪存使用寿命。有两个方式你可以观察到写放大。第一个方式是读取DB::GetProperty("rocksdb.stats", &stats)的输出。第二个是使用你的DB写速率除以你的磁盘写带宽。

**读放大**是每秒磁盘读的数量。如果你需要读5个页来响应一个查询，读放大就是5。逻辑读是从缓存得到的数据，要么从Rocksdb的块缓存，要么从OS的文件缓存。物理读通过存储设备，闪存或者硬盘，处理。逻辑读比物理读便宜很多，但是会导致CPU开销。你也可以通过iostat的输出估算读放大，但是这个结果包含了查询和压缩的读。

**空间放大**是数据库磁盘上的文件的大小和数据大小的比。如果你Put 10MB的数据到数据库，它使用了100MB的磁盘，那么空间放大为10.你通常希望设置一个硬性限制给空间放大，这样你就不会吧磁盘空间或者内存用光了。

为了了解这三个放大因子在不同数据库算法下的情况，我们强烈推荐[Mark Callaghan关于高并发的演讲](http://vimeo.com/album/2920922/video/98428203)

# Rocksdb统计

当调试性能的时候，有一些工具可以帮助到你：

**statistics** —— 把这个设置给rocksdb::CreateDBStatistics()。任何时候，通过调用options.statistics.ToString()，你可以得到一个人类可读的Rocksdb统计信息。参考[统计](Statistics.md)了解更多信息。

**stats_dump_period_sec** ——我们每stats_dump_period_sec秒就会把统计信息导出到日志文件。默认为600，意味着每10分钟导出一次。你可以在应用里调用db->GetProperty("rocksdb.stats")得到相同的数据。

每db->GetProperty("rocksdb.stats")，你会在日志文件里找到这样的数据：

```
** Compaction Stats **
Level Files  Size(MB) Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) Comp(cnt) Avg(sec) Stall(sec) Stall(cnt) Avg(ms)     KeyIn   KeyDrop
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
L0      2/0        15   0.5     0.0     0.0      0.0      32.8     32.8       0.0   0.0      0.0     23.0    1457      4346    0.335       0.00          0    0.00             0        0
L1     22/0       125   1.0   163.7    32.8    130.9     165.5     34.6       0.0   5.1     25.6     25.9    6549      1086    6.031       0.00          0    0.00    1287667342        0
L2    227/0      1276   1.0   262.7    34.4    228.4     262.7     34.3       0.1   7.6     26.0     26.0   10344      4137    2.500       0.00          0    0.00    1023585700        0
L3   1634/0     12794   1.0   259.7    31.7    228.1     254.1     26.1       1.5   8.0     20.8     20.4   12787      3758    3.403       0.00          0    0.00    1128138363        0
L4   1819/0     15132   0.1     3.9     2.0      2.0       3.6      1.6      13.1   1.8     20.1     18.4     201       206    0.974       0.00          0    0.00      91486994        0
Sum  3704/0     29342   0.0   690.1   100.8    589.3     718.7    129.4      14.8  21.9     22.5     23.5   31338     13533    2.316       0.00          0    0.00    3530878399        0
Int     0/0         0   0.0     2.1     0.3      1.8       2.2      0.4       0.0  24.3     24.0     24.9      91        42    2.164       0.00          0    0.00      11718977        0
Flush(GB): accumulative 32.786, interval 0.091
Stalls(secs): 0.000 level0_slowdown, 0.000 level0_numfiles, 0.000 memtable_compaction, 0.000 leveln_slowdown_soft, 0.000 leveln_slowdown_hard
Stalls(count): 0 level0_slowdown, 0 level0_numfiles, 0 memtable_compaction, 0 leveln_slowdown_soft, 0 leveln_slowdown_hard

** DB Stats **
Uptime(secs): 128748.3 total, 300.1 interval
Cumulative writes: 1288457363 writes, 14173030838 keys, 357293118 batches, 3.6 writes per batch, 3055.92 GB user ingest, stall micros: 7067721262
Cumulative WAL: 1251702527 writes, 357293117 syncs, 3.50 writes per sync, 3055.92 GB written
Interval writes: 3621943 writes, 39841373 keys, 1013611 batches, 3.6 writes per batch, 8797.4 MB user ingest, stall micros: 112418835
Interval WAL: 3511027 writes, 1013611 syncs, 3.46 writes per sync, 8.59 MB written

```

# 压缩信息

在level N和level N+1之间执行的压缩流程的压缩信息会在level N+1处（输出层）进行汇报。这里是一个快速参考：

- level —— leveled压缩在LSM中的层。对于universal压缩，所有文件都在L0.**Sum**有所有层的数据的和。**Int**类似于**Sum**但是只限于从上一次汇报之后的间隔之间的数据。
- Files —— 他有两个如(a/b)的数值。第一个数字是这一层的文件数量。第二个是当前该层正在进行压缩的文件的数量。
- Score —— 除了L0之外的层，score是（当前层大小）/(最大层大小)。值为0或者1都是正常的，但是如果值大于1，意味着这个层需要被压缩。对于L0，score根据当前文件的数量和触发压缩的文件数量来计算。
- Read(GB) —— 在level N和level N+1之间压缩的时候读取的总字节数。这包括了从level N和level N+1读取的数据。
- Rn(GB)：在level N和level N+1之间压缩的时候，从Level N读取的字节数。
- Rnp1(GB)：在level N和level N+1之间压缩的时候，从Level N+1读取的字节数。
- Write(GB)：在level N和level N+1之间压缩的时候写出的总字节数。
- Wnew(GB)：写到level N+1的新字节数，计算方式为：(写到N+1的总字节数) - (与level N 压缩的时候，从N+1读取的字节数)
- Moved(GB)：压缩期间移动到Level N+1的字节数。这个场景下，没有任何IO发生，除了更新manifest以指示原本在level X的文件，现在在level Y了 以外。
- W-Amp：（写入到LevelN+1的总字节数） / (从levelN读取的字节数)。这是从Level N到Level N+1的写放大
- Rd(MB/s)：从Level N和level N+1读取的数据的速度。通过  (Read(GB) * 1024) / 压缩时间 计算得到。
- Wr(MB/s)：从Level N和level N+1写数据的速度。参考Rd(MB/s)。
- Rn(cnt)：在level N和level N+1之间压缩的时候，从Level N读取的总文件数量。
- Rnp1(cnt)：在level N和level N+1之间压缩的时候，从Level N+1读取的总文件数量。
- Wnp1(cnt)：在level N和level N+1之间压缩的时候，写入Level N+1的文件数量。
- Wnew(cnt)：(Wnp1(cnt) - Rnp1(cnt)) —— 作为level N和level N+1之间压缩的结果，增加的文件的数量。
- Comp(sec)：在level N和level N+1之间压缩花费的总时间
- Comp(cnt)：在level N和level N+1之间压缩发生的压缩次数
- Avg(sec)：在level N和level N+1之间压缩，每次压缩的平均时间。
- Stall(sec)：由于level N+1没有被压缩（压缩score很高）而导致的写失速总时间。
- Stall(cnt)：由于level N+1没有被压缩而导致的写失速总次数。
- Avg(ms)：由于level N+1没有被压缩而导致的写失速的平均时间，单位毫秒。
- KeyIn：压缩过程中压缩的key的数量
- KeyDrop：压缩过程中丢弃的key（没有被写出）的数量。

# 通用信息

每层的压缩信息之后，我们同时输出一些通用信息。通用信息会报告**累计**信息和**间隔**信息。累计信息报告从Rocksdb实例打开到现在的总数据。间隔信息报告从上一次信息输出到现在的间隔中间的信息。

- Uptime(secs) ： total —— 这个实例跑的时间。interval —— 上次信息导出之后过了多少秒。
- Cumulative/Interval writes：total —— Put调用数量；keys —— Put调用中，WriteBatches 的项目量；batches —— 群提交的数量，每个群提交持久化一个或者多个Put调用（他们并行发生，一个时间点会有一个以上的Put调用被持久化）；per batch —— 一个batch的字节数的平均数量；ingest —— 写入DB的总字节数（不计算压缩）；stall micro —— 由于压缩落后导致的写失速的微秒时间。
- Cumulative/Interval WAL：writes —— 记录在WAL的写数量；syncs —— fsync或者fdatasync被调用的次数；write per sync —— 写数量和sync的比例；GB written —— 写入WAL的GB数量。
- Stalls：从开始到现在，所有写失速类型导致的写失速的总时间，单位秒：level0_slowdown —— 由于level0_slowdown_writes_trigger导致的写失速。level0_numfiles —— 由于level0_stop_writes_trigger导致的写失速。memtable_compaction —— 由于所有metable都写满导致的写失速，落盘速度跟不上。leveln_slowdown —— 由于soft_rate_limit和hard_rate_limit导致的写失速。

# 性能上下文和IO信息上下文

[性能上下文和IO信息上下文](Perf-Context-and-IO-Stats-Context.md)可以帮助我们了解一个特定查询的情况。

# 并发选项

在LSM架构，有两个后台线程：落盘和压缩。两个都可以通过线程并行执行，以发挥存储技术的并行性能。落盘线程在 高优先池，而压缩线程在低优先池。为了增加每个池的线程数，可以调用

```
 options.env->SetBackgroundThreads(num_threads, Env::Priority::HIGH);
 options.env->SetBackgroundThreads(num_threads, Env::Priority::LOW);
```

为了从更多的线程获得收益，你可能需要修改并行压缩的压缩和落盘线程为最大数量：

**max_background_compactions**为后台压缩的最大线程数。默认为1，但是为了完全利用CPU和存储，你可能会希望增加这个到接近系统的核的数量。

**max_background_flushes**为落盘并发数。通常设置为1就足够了。

## 通用选项。

**filter_policy** —— 如果你需要做点查询你一定希望打开bloom过滤器。我们使用bloom过滤器来避免不必要的磁盘访问。你应该把filter_policy赋值给rocksdb::NewBloomFilterPolicy(bits_per_key)。默认bits_per_key 为10，带来袋盖1%的假阳性率。更大的bits_per_key会降低假阳性率，但是增加内存使用和空间放大。

**block_cache** —— 我们通常推荐把这个设置赋值给rocksdb::NewLRUCache(cache_capacity, shard_bits)的结果。块缓存缓存了未压缩的块。另一方面，OS缓存，缓存了压缩了的块（因为他们是以这种方式存储在文件的）。因此，同时使用block_cache和OS缓存是合理的。我们需要对块缓存的访问上锁，并且有时候我们看到RockDB在块缓存互斥锁上有瓶颈，特别是当DB的大小小于RAM的时候。在这种情况，设置shard_bits为一个更大的数字，把块缓存分片就很合理了。如果shard_bits为4，分片数量为16。

**allow_os_buffer** —— 如果为false，我们不会把文件缓存在OS的缓存。查看上面的注释。

**max_open_files** —— RocksDB会保存所有文件描述符到一个表缓存。如果文件描述符的数量超过了max_open_files，一些文件会从表缓存中被淘汰，并且他们的文件描述符会被关闭。这意味着每个读取必须遍历表缓存以找到他需要的文件。设置max_open_files为-1以永远允许打开文件，可以避免昂贵的表缓存调用。

**table_cache_numshardbits** —— 这个选项控制表缓存分片。如果表缓存互斥锁竞争激烈，增加这个。

**block_size** —— RocksDB把用户数据打包到块里。当尝试从一个表文件一个键值对的时候，一个块项目会被载入内存。块大小默认为4KB。每个表文件包含一个索引，罗列了所有块的偏移。增加block_size意味着索引会包含更少的项（因为每个文件的块少了），因此索引会更小。增加block_size会减少内存使用，和空间放大，但是会带来读放大。

# 缓存分片和线程池

有时候你可能希望在一个进程里跑多个RocksDB实例。RocksDB提供一个方式让这些实例共享块缓存和线程池。为了共享块缓存，给所有实例赋值同一个缓存对象。

```
first_instance_options.block_cache = second_instance_options.block_cache = rocksdb::NewLRUCache(1GB)
```

这会是两个实例共享一个1GB的块缓存。

线程池与Env对象结合。当你构造Options的时候，options.env被设置为Env::Default()，通常情况下这都是最好的。由于所有的Options使用同一个静态对象Env::Default()，线程池默认就是共享的。参考[并发选项](#并发选项)以了解如何设置线程池的线程数量。这样，你可以设置最大并行运行的压缩和落盘，即使运行多个RocksDB实例。

# 落盘选项

所有写入到RocksDB的都是先插入一个名为memtable的内存数据结构。一旦**活跃的memtable**满了，我们创建一个新的，然后标记旧的为只读。我们成只读的memtable为**不可修改**。在任何时候，都刚好只有一个活跃的memtable，然后又0个或者更多的不可修改memtable。不可修改memtable总是等待被落盘到存储。有三个选项控制落盘行为。

**write_buffer_size** 设置一个单独memtable的大小。一旦memtable超过这个大小，他就会被标记为不可修改并且一个新的会被创建。

**max_write_buffer_number**设置memtable的最大数量，活跃和不可修改加在一起。如果活跃memtable填满了，然后总memtable的数量大于max_write_buffer_number，我们会让后续的写入失速。在落盘进程慢于写入速度的时候，就会发生。

**min_write_buffer_number_to_merge**是落盘前需要合并的memtable的最小数量。例如，如果选项设置为2，不可修改memtable只会在有两个的时候落盘 —— 一个单一的不可修改memtable绝对不会落盘。如果多个memtable被合并到一起，会有更少的数据被写入存储，因为两个更新被合并到一个单独的key。然而，每个Get()必须线性遍历所有不可修改的memtable已检查是否有key存在。把这个值设置的太高可能会伤害性能。

例子：选项为：

```
write_buffer_size = 512MB;
max_write_buffer_number = 5;
min_write_buffer_number_to_merge = 2;
```

如果写入速率为16MB/s。在这个例子，一个新的memtable会每32秒创建一次，然后两个memtable会被合并到一起然后每64秒落盘一次。根据工作集合的大小，落盘大小会在512MB到1GB之间。为了防止落盘无法跟上写速度，memtable使用的内存大小被限制为 5*512MB = 2.5GB。当这个值达到了，后续写入会被拦截，知道落盘结束，并且memtable使用的内存被释放。

# Level风格压缩

在Level风格压缩，数据库文件按层组织。memtable被落盘到level 0的文件，那里包含了最新的数据。更高层包含更老的数据。level 0 的文件会有交叉，但是在level 1 和 更高的没有交叉。结果，Get通常需要检查level 0的每个文件，但是对于后续的层，不会超过一个文件包含这个key。每个层都10倍（这个因数是可配置的）大于之前一层。

一次压缩可能携带一些在level N的文件，然后与level N+1的有交叉的文件进行压缩。两个在不同层的压缩操作或者不同key范围的操作可以相互独立进行或者并发进行。压缩速度直接与最大写速率成比例。如果压缩不能跟上写速率，数据库使用的空间会持续增长。以这种方式配置RocksDB使他能以高并发执行压缩，完全利用存储的性能**非常重要**。

Level 0 和 1 的压缩有点取巧。level 0 的文件通常覆盖整个key空间。当压缩L0 -> L1（从level 0 到 level 1），压缩包含所有Level 1的文件。将所有L1的文件与L0压缩，则L1 -> L2的压缩无法同时进行；他必须等到L0 -> L1 的压缩结束。如果 L0 -> L1压缩很慢，他会变成系统内大部分时间里唯一运行的压缩，因为其他的压缩必须等待他完成。

L0 -> L1 压缩同样是单线程的。很难在单线程压缩中得到一个好的吞吐。为了检查是不是这里出了问题，检查磁盘利用率。如果磁盘不是完全被利用起来，可能压缩配置有问题。我们通常推荐 通过**设置L0跟L1的大小差不多** 以达到 尽快完成L0 -> L1压缩的目的。

一旦你决定了Level 1 的合适大小，你必须决定层乘数因子。假设你的level 1大小为512MB，层乘数因子为10，并且数据库的大小为500GB。Level 2 的大小就是5GB，level 3 51GB，level 4 512GB。因为你的数据库大小为500GB，level 5以及更高的层会是空的。

空间放大很好计算。为(512 MB + 512 MB + 5GB + 51GB + 512GB) / (500GB) = 1.14。这里是我们如何计算写放大：每个字节先会写到Level 0。之后被压缩到Level 1.因为Level 1的大小跟Level 0 相同，从L0 -> L1压缩的写放大为 2。然而，当一个从Level 1 来的字节压缩到Level 2的时候，他与level 2的10个byte压缩（因为level 2 是10x倍大）。L2 -> L3和L3 -> L4也是一样。

因此，总写放大接近 1 + 2 + 10 + 10 + 10 = 33。点查询必须查询level 0 的所有文件然后每一层最多查询一次。然而，bloom过滤器可以帮我们极大减少读放大。不过，短期存活的区间扫描会有点昂贵。Bloom过滤器在区间扫描的时候没什么用，所以读放大为number_of_level0_files + number_of_non_empty_levels。

现在我们深入探讨控制level压缩的选项。我们会从更重要的开始。

**level0_file_num_compaction_trigger** —— 一旦level 0 的文件数量达到这个值，L0->L1压缩就会触发。我们可以这样估算level 0在稳定状态的大小：write_buffer_size * min_write_buffer_number_to_merge * level0_file_num_compaction_trigger。

**max_bytes_for_level_base**和**max_bytes_for_level_multiplier** —— max_bytes_for_level_base是一个Level 1的总大小。就如之前说的，我们推荐这个跟level 0的大小接近。每个后续层为max_bytes_for_level_multiplier倍于前一个。默认为10，我们不推荐修改他。

**target_file_size_base** 和 **target_file_size_multiplier** —— 在level 1的文件大小为target_file_size_base字节。每下一层的文件大小会是target_file_size_multiplier倍大于前一层。然而，默认target_file_size_multiplier为1，所以每一层文件的大小都一样大，这通常是个好事。我们推荐设置target_file_size_base为max_bytes_for_level_base/10，这样我们在level 1就有10个文件。

**compression_per_level** —— 使用这个选项来设置不同层的压缩风格。通常我们不压缩level 0 和level 1，值在更高的层压缩数据。你甚至可以再最高层设置最慢的压缩算法，在最底层设置更快的压缩算法（最高层为Lmax）。

**num_levels** —— num_levels比预期的数据库的层数高是安全的。一些更高的层会是空的，但是这不会影响数据库的性能。只有当你希望你的层数大于7（默认值）的时候才修改这个选项。

# Universal压缩

level风格压缩在某些场景会有很高的写放大。对于写多的场景，你可能会因为磁盘推图而遇到瓶颈。为了优化这些场景，RocksDB引入了一个新的压缩风格，我们称之为Universal压缩，希望减少写放大。然而，这可能增加读放大，并且总是增加空间放大。**Universal压缩有大小限制。当你的DB（或者列族）大于100GB的时候，请注意**。参考[Universal压缩](Universal-Compaction.md)了解细节。

使用universal压缩，一个压缩流程可能张女士增加2的空间放大。换句话说，如果你存储10GB的数据在数据库，压缩过程会消耗额外的10GB，还要加入额外的空间放大。

然而，当有技术可以帮助我们减少临时的内存翻倍。如果你使用universal压缩，我们强烈你分片数据，并且放置在多个RocksDB实例。假设你有S个分片。然后配置Env线程池，只使用N个压缩线程。只有N个分片，S个线程会有额外的空间放大，因此得到N/S的额外放大，而不是1。例如，如果你的DB是10GB，并且你配置100个分片，每个分片会有100MB的数据。如果你配置你的线程池为20个并发压缩，你会只需要额外的2GB数据，而不是10GB。同事，压缩会并行执行，可以完全利用你的存储并发性能。

**max_size_amplification_percent** —— 大小放大，定义为存储数据库一个byte数据额外需要的存储（百分比）。默认为200，意味着一个100byte的数据库可以获取300byte的存储空间。300byte中的200 byte只在压缩过程中暂时用到。增加这个限制减小写放大，但是（显然）增加空间放大。

**compression_size_percent** —— 数据库中压缩的数据的比例。较老的数据会被压缩，更新的数据不会被压缩。如果设置为-1（默认），所有数据都会被压缩。减小compression_size_percent会减少CPU使用率，增加空间放大。

参考[Universal压缩](Universal-Compaction.md)了解更多信息

# 写失速

参考[写失速](https://rocksdb.org.cn/doc/Write-Stalls.html)了解更多细节

# 前缀数据库

RocksDB保持所有排序号并且支持顺序迭代。然而，有些应用不需要key为完全排序。他们只关心一个固定前缀的key的排序。

这些应用可以从prefix_extractor中得到好处。

**prefix_extractor** —— 一个SliceTransform对象，定义key前缀。key前缀之后被用于实现一些有趣的优化：

定义bloom过滤器，可以减少前缀区间查询的读放大（比如，给我所有以前缀XXX开头的key）。确保定义**Options::filter_policy**。

使用基于哈希表的memtable以避免memtable里二分搜索的开销。

给表文件增加哈希索引以避免表文件中二分搜索的开销。对于(2)和(3)的细节，参考[自定义memtable和表工厂](https://rocksdb.org.cn/doc/Basic-Operations.html#memtable-and-table-factories)。请注意，(1)通常已经降低足够的IO了。（2）和（3）可以在某些场景降低CPU开销，并且通常带来一些内存开销。你应该只在CPU为你的瓶颈，并且没有其他更简单的调优手段的时候尝试他们，毕竟这不是通用尝试。确保查看了include/rocksdb/options.h中的关于prefix_extractor的注释。

# Bloom过滤器

Bloom过滤器是基于可能性的数据结构，用于检测一个元素是不是存在于一个结合中。RocksDB中的Bloom过滤器通过一个名为*filter_polic*的选项控制。当一个用户调用Get(key)，会有一个文件列表，可能包含这个key。通常是Level 0的所有文件，以及大于0的每一层中的一个文件。然而，在我们读取每个文件前，我们先咨询bloom过滤器。Bloom过滤器会过滤掉大部分不包含该key的文件的读取。在大多数时候，Get通常只会做一次文件读取。Bloom过滤器总是保持在内存中，以方便打开文件，除非BlockBasedTableOptions::cache_index_and_filter_blocks为true。打开的文件的数量通过max_open_files选项控制。

有两个bloom过滤器类型：基于块的，和全过滤。

## 基于块的过滤器

通过调用一下接口使用基于块的过滤器：

```
options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10, true))
```

基于块的bloom过滤器是根据每个块分别建立的。在一个读取中，我们先咨询一个索引，返回我们正在找的块。现在我们有一个块了，我们咨询bloom过滤器来过滤这个块。

## 全过滤

通过一下调用设置全过滤：

```
options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10, false))
```

全过滤针对每个文件构建。每个文件只有一个bloom过滤器，这意味着我们可以先查询bloom过滤器，而不用查询索引。如果key不在bloom过滤器，相比基于块的过滤器，我们省略一个索引搜索。

全过滤可以进一步分片 : [分片过滤](Partitioned-Index-Filters.md)

# 自定义memtable和表格式

高级用户可以配置自定义的memtable和表格式

**memtable_factory** —— 定义memtable。这里是我们支持的memtable：

- SkipList —— 默认的memtable
- HashSkipList —— 只能与prefix_extractor工作。他把key放入基于key前缀的桶中。每个桶是一个skiplist。
- HashLinkedList —— 只能与prefix_extractor工作。他把key放入基于key前缀的桶中。每个桶是一个linked list。

**table_factory** —— 定义表格式。这里是我们支持的表格式：

- 基于块 —— 这是默认的表。适合于磁盘和闪盘上排序好的数据。他根据块的大小分块定位和加载（参考block_size选项）。因此成为基于块。
- 平表 —— 只能与prefix_extractor一起工作。适用于在内存中排序好的数据（在tmpfs文件系统）。可以按byte定位。

# 内存使用

为了了解rocksdb是如何使用内存的，参考另一个wiki页[内存使用](Memory-usage-in-RocksDB.md)

# 机械硬盘的差异

在机械硬盘上，内存/持久化存储速比率常会低很多。如果数据和RAM的比率如果比较大，那么你可以减少对性能要求很高的数据需要的内存，以保证重要的数据在RAM。建议：

- 使用相对**更大的块大小**以减少索引块的大小。你应该使用至少64KB的块大小。你可以考虑256KB甚至512KB。使用大块带来的问题是RAM被块缓存浪费了。
- 打开**BlockBasedTableOptions.cache_index_and_filter_blocks=true**因为通常你不能把所有索引和bloom过滤器放入内存。即使你可以，也可以为了安全起见，打开这个。
- **打开options.optimize_filters_for_hits**以减少一些bloom过滤器块大小。
- 小心确保你有足够的内存来保存所有的bloom过滤器。如果你不能，那么bloom过滤器可能会损害性能。
- 尝试**尽量紧凑的key编码**。更短的key可以减小索引块大小。

与闪存相比，机械硬盘通常提供更低的随机读吞吐。

- 设置**options.skip_stats_update_on_db_open=true**以加快DB打开时间。
- 这是一个有争议的建议：使用**基于level的压缩**，因为他对于减少磁盘读更友好
- 如果你使用基于level的压缩，使用**options.level_compaction_dynamic_level_bytes=true**。
- 如果服务器有多个硬盘，设置**options.max_file_opening_threads**为一个大于1的值。

随机读和序列化读的吞吐量差在机械磁盘上会比较大。建议：

- 为压缩的输入，打开RocksDB层的预读取：**options.compaction_readahead_size**和**options.new_table_reader_for_compaction_inputs=true**
- 使用相对**大文件尺寸**，我们推荐至少256MB。
- 使用相对大的块大小。

机械磁盘通常比闪存大：

- 为了避免过多的文件描述符，使用更大的文件。我们推荐文件大小至少256MB。
- 如果你使用universal风格压缩，不要令单个DB大小太大，因为全压缩会花费大量时间，并且影响性能。你可以使用更多的DB实例，单个DB的大小应该小于500GB。

# 示例配置

在这一节，我们会展现一些我们在生产环境上的RocksDB配置。

## 闪存上的前缀数据库

这个服务使用RocksDB来实现前缀区间搜索和点查询。在闪存上运行。

```
 options.prefix_extractor.reset(new CustomPrefixExtractor());
```

由于服务不需要读完整的顺序迭代（参考[前缀数据库](#前缀数据库)），我们定义前缀提取器。

```
rocksdb::BlockBasedTableOptions table_options;
 table_options.index_type = rocksdb::BlockBasedTableOptions::kHashSearch;
 table_options.block_size = 4 * 1024;
 options.table_factory.reset(NewBlockBasedTableFactory(table_options));
```

我们在表文件中使用一个哈希索引以加快前缀查找，但是这增加存储空间和内存使用。

```
 options.compression = rocksdb::kLZ4Compression;
```

LZ4压缩减少了CPU使用，但是增加存储空间。

```
 options.max_open_files = -1;
```

这个设定关闭在表缓存中搜索文件，因此加快所有查询。如果你的服务的打开文件数非常高，这总是一个好的设定。

```
 options.options.compaction_style = kCompactionStyleLevel;
 options.level0_file_num_compaction_trigger = 10;
 options.level0_slowdown_writes_trigger = 20;
 options.level0_stop_writes_trigger = 40;
 options.write_buffer_size = 64 * 1024 * 1024;
 options.target_file_size_base = 64 * 1024 * 1024;
 options.max_bytes_for_level_base = 512 * 1024 * 1024;
```

我们使用level风格的压缩。Memtable的大小为64MB并且周期性落盘到Level 0.压缩 L0 -> L1在Level 0 有 10个文件的时候触发（总共640MB）。当L0有640MB，压缩触发，压入L1，最大的大小是512MB，总DB大小？？？

```
 options.max_background_compactions = 1
 options.max_background_flushes = 1
```

任何时候，只能有1个并发压缩和1个落盘线程在进行。然而，系统有多个分片，所以在不同分片会有多个压缩。否则，只有两个线程往存储写入数据，利用率很低。

```
 options.memtable_prefix_bloom_bits = 1024 * 1024 * 8;
```

使用memtable的bloom过滤器，一些memtable的访问可以避免。

```
options.block_cache = rocksdb::NewLRUCache(512 * 1024 * 1024, 8);
```

块缓存被配置为512MB。（这个在好几个分片共享？）

## 全排序数据库，闪存。

这个数据库同事执行Get和全排序迭代。分片？？？

```
options.env->SetBackgroundThreads(4);
```

我们先设置4个线程到线程池。

```
options.options.compaction_style = kCompactionStyleLevel;
options.write_buffer_size = 67108864; // 64MB
options.max_write_buffer_number = 3;
options.target_file_size_base = 67108864; // 64MB
options.max_background_compactions = 4;
options.level0_file_num_compaction_trigger = 8;
options.level0_slowdown_writes_trigger = 17;
options.level0_stop_writes_trigger = 24;
options.num_levels = 4;
options.max_bytes_for_level_base = 536870912; // 512MB
options.max_bytes_for_level_multiplier = 8;
```

我们使用level风格压缩，高并发。memtable大小为64MB，level0文件数量为8。这意味着压缩在L0的数据增长到512MB的时候触发。L1的大小为512MB，每个层8倍大于上一层，L2 4Gb，L3 32GB。

## 机械硬盘上的数据库

即将到来。。。

## 完整功能的内存数据库

在这个例子，数据库被挂载到了tmpfs文件系统。

使用mmap读：

```
options.allow_mmap_reads = true;
```

禁止块缓存，打开bloom过滤器，减少重启的开销：

```
BlockBasedTableOptions table_options;
table_options.filter_policy.reset(NewBloomFilterPolicy(10, true));
table_options.no_block_cache = true;
table_options.block_restart_interval = 4;
options.table_factory.reset(NewBlockBasedTableFactory(table_options));
```

如果你希望优先考虑速度，你可以关闭压缩：

```
options.compression = rocksdb::CompressionType::kNoCompression;
```

否则，打开一个轻量压缩，LZ4或者Snappy。

设置更激进的压缩方式，并且为落盘和压缩分配更多的线程。

```
options.level0_file_num_compaction_trigger = 1;
options.max_background_flushes = 8;
options.max_background_compactions = 8;
options.max_subcompactions = 4;
```

保持所有文件打开：

```
options.max_open_files = -1;
```

当读取数据的时候，考虑设置ReadOptions.verify_checksums = false。

## 内存前缀数据库

在这个例子，数据库挂载在tmpfs文件系统。我们使用自定义的格式来加速，一些其他功能无法支持。我们只支持Get和前缀范围搜索。WAL日志被排序好并且存在硬盘，以避免消耗非用于查询的内存。不支持Prev。

由于数据库是在内存，我们不关心写放大。我们更关心读放大和空间放大。这是一个有趣的例子，因为我们对压缩调优到极致，所以通常只有一个SST表存在于系统。因此我们减少了读和空间放大，而写放大很大。

由于使用universal压缩，压缩期间，我们的硬盘空间会高效地翻倍。这对内存数据库非常危险。因此我们把数据分片城400个RocksDB实例。我们只允许两个并发压缩，所以只有两个分片会使存储翻倍。

在这个例子，前缀哈希可以用于允许系统使用哈希索引，而不是二分搜索，同时，如果可能，迭代的时候打开bloom过滤器：

```
options.prefix_extractor.reset(new CustomPrefixExtractor());
```

使用为了低延迟构建的内存定位表格式，需要mmap模式打开：

```
options.table_factory = std::shared_ptr<rocksdb::TableFactory>(rocksdb::NewPlainTableFactory(0, 8, 0.85));
options.allow_mmap_reads = true;
options.allow_mmap_writes = false;
```

使用哈希链表memtable以使用memtable的哈希索引：

```
options.memtable_factory.reset(rocksdb::NewHashLinkListRepFactory(200000));
```

当从memtable读取数据的时候，为哈希表打开bloom过滤器以减少内存访问（通常意味着CPU缓存未命中），以防止key在memtable中不存在。

```
options.memtable_prefix_bloom_bits = 10000000;
options.memtable_prefix_bloom_probes = 6;
```

对压缩调优，一个全量压缩会在有两个文件的时候马上开始。我们hack了universal压缩的参数：

```
options.compaction_style = kUniversalCompaction;
options.compaction_options_universal.size_ratio = 10;
options.compaction_options_universal.min_merge_width = 2;
options.compaction_options_universal.max_size_amplification_percent = 1;
options.level0_file_num_compaction_trigger = 1;
options.level0_slowdown_writes_trigger = 8;
options.level0_stop_writes_trigger = 16;
```

调优bloom过滤器以最小化内存访问：

```
options.bloom_locality = 1;
```

所有表的读者对象总是被缓存，避免读取的时候表缓存访问：

```
options.max_open_files = -1;
```

同一时间使用一个memtable。他的大小根据我们希望的压缩间隔来决定。我们调优压缩，所以每次落盘后，一个全量压缩都会触发，消耗CPU。memtable越大，压缩间隔会越大，同时，我们看到内存效率更低，更差的查询性能和重启时更长的恢复时间：

```
options.write_buffer_size = 32 << 20;
options.max_write_buffer_number = 2;
options.min_write_buffer_number_to_merge = 1;
```

多个DB实例共享两个压缩线程：

```
options.max_background_compactions = 1;
options.max_background_flushes = 1;
options.env->SetBackgroundThreads(1, rocksdb::Env::Priority::HIGH);
options.env->SetBackgroundThreads(2, rocksdb::Env::Priority::LOW);
```

设置WAL：

```
options.bytes_per_sync = 2 << 20;
```

## 对于内存块表的建议

**hash_index**：在新的版本，哈希索引对基于块的表打开。他会使用5%的额外存储空间，但是随机读取比普通二分搜索快50%。

table_options.index_type = rocksdb::BlockBasedTableOptions::kHashSearch;


**block_size**：默认，这个值为4K。如果压缩被打开，一个更小的块大小会导致更高的随机度速度，因为解压缩的开销减小了。但是块大小不能太小，否则压缩就不起作用了。推荐设置到1k。

**verify_checksum**：由于我们在tmpfs上排序好，并且关心读性能，校验和会被关闭。

# 最后的考虑

很不幸，最优化配置RocksDB不可忽略。即使是我们作为RocksDB开发者也不能完全明白每种配置的作用。如果你希望完全针对你的工作环境优化RocksDB，我们推荐实验和压力测试，同事注意三个放大因子。同事，请不要犹豫到[RocksDB开发者讨论组](https://www.facebook.com/groups/rocksdb.dev/)寻找我们的帮助。


