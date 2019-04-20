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

**filter_policy**

