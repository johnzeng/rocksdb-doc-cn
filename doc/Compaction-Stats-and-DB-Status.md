# 压缩统计和数据库状态

哪里开启他？你可以从以下路径找到压缩统计:

1、RocksDB每*stats_dump_period_sec* 秒导出统计信息到LOG文件中，默认600秒，也就意味着统计信息会每10分钟写入LOG文件。

2、你可以在应用中通过db->GetProperty("rocksdb.stats")得到相同的（统计）数据。

两种方法的输出都像这样：

```c++
Compaction Stats
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

DB Stats
Uptime(secs): 128748.3 total, 300.1 interval
Cumulative writes: 1288457363 writes, 14173030838 keys, 357293118 batches, 3.6 writes per batch, 3055.92 GB user ingest, stall micros: 7067721262
Cumulative WAL: 1251702527 writes, 357293117 syncs, 3.50 writes per sync, 3055.92 GB written
Interval writes: 3621943 writes, 39841373 keys, 1013611 batches, 3.6 writes per batch, 8797.4 MB user ingest, stall micros: 112418835
Interval WAL: 3511027 writes, 1013611 syncs, 3.46 writes per sync, 8.59 MB written
```



## 压缩统计

压缩统计在压缩N层和N+1层且输出到N+1层时进行。

下面是快速参考：

- Level: 表示层级压缩的层级。 对于universal compaction，所有文件都在L0层。 **Sum** 是所有层的数据. **Int** 像 **Sum** 一样是聚合数据，但数据限制在最近一个报告周期。
- Files: 有两个值 (a/b). 第一个值是在这个层级上的文件书. 第二个是正在这个层级上做压缩的文件数。
- Score: 除了L0外的分数为 (当前层级的大小) / (层级的最大大小). 只要大于1，就意味着该层级需要呗压缩。对L0来说分数是当前的文件数和压缩触发值的比。
- Read(GB): 层级N和N+1的压缩中读取数据的字节总数。包括从N层读的数据和N+1层读的数据。
- Rn(GB): 层级N和N+1的压缩中读取N层的字节数。
- Rnp1(GB): 层级N和N+1的压缩中读取N+1层的字节数
- Write(GB): 层级N和N+1的压缩中写入数据的字节总数。
- Wnew(GB): 写入N+1层的新数据的总字节数, 通过 (写入 N+1层的总字节数) - (压缩时从N+1层读取的字节总数)计算。
- Moved(GB): 压缩时移入N+1层的字节总数. 这种情况下除了更新manifest这个文件从X层到Y层外，没有任何IO。
- W-Amp: (写入N+1层的总字节数) / (从N层读取的总字节数). 这是N层到N+1层的写放大。
- Rd(MB/s): 压缩N到N+1层时的读取速度。 通过 (Read(GB) * 1024) / compaction周期计算得出。
- Wr(MB/s): 压缩N到N+1层时的写入速度。 见 Rd(MB/s)。
- Rn(cnt): 压缩N到N+1层时从N层读取的文件总数。
- Rnp1(cnt): 压缩N到N+1层时从N+1层读取的文件总数。
- Wnp1(cnt): 压缩N到N+1层时写入N+1层的文件总数。
- Wnew(cnt): (Wnp1(cnt) - Rnp1(cnt)) -- 压缩N到N+1层时增长的文件数
- Comp(sec): 压缩N到N+1层花费的时间数。
- Comp(cnt): 压缩N到N+1层的压缩任务数。
- Avg(sec): 压缩N到N+1层每个压缩任务的平均耗时。
- Stall(sec): 因为N+1层未压缩而导致的写暂停时间（压缩分数高于1)。
- Stall(cnt): 因为N+1层未压缩而导致的写暂停数。
- Avg(ms): 因为N+1层未压缩而导致的一次写入被暂停的平均时间。
- KeyIn: 压缩期间写入的Key数。
- KeyDrop: 压缩期间被丢弃的Key数。



### 通用统计

在每层的压缩统计之后，我们也输出一些通用的统计。通用统计报告累计数据和定期数据。累计数据报告从RocksDB启动后的数据总值。定期数据则报告从上一次统计结束后开始的数据。

- Uptime(secs): total -- 从实例启动后运行的总时间, interval -- 从上一次统计数据导出后经过的时间。

- Cumulative/Interval writes: total -- 总Put调用数; keys -- Put调用中包含的总key数; batches --写入的batch数 (并发下可能一个batch内有多个put请求); per batch -- 单个batch的平均大小; ingest -- 写入 DB的总大小 (不包括compactions); stall micros - 因后端compaction导致的写入暂停微秒数。

- Cumulative/Interval WAL: writes -- 写入WAL的写请求数; syncs - fsync 或 fdatasync被调用次数; writes per sync - 每次syncs包含的写入请求数; GB written - 写入WAL的GB大小

- Stalls: 每一种暂停类型的总数和时间

  level0_slowdown -- 因为 `level0_slowdown_writes_trigger`触发暂停。

  level0_numfiles --   因为 `level0_stop_writes_trigger` 触发暂停。

  memtable_compaction -- 因为memtables数量满，flush进程无法继续触发暂停。

  leveln_slowdown -- 因为 `soft_rate_limit` and `hard_rate_limit`触发暂停。