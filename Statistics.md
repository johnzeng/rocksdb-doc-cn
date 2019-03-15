DB的Statistics提供一个历史累计数据。他提供来自DB属性以及[性能和IO状态上下文]()的不同的功能:历史性能累计统计，DB属性提供当前状态；DB统计提供一个针对所有操作的聚合的试图，而性能和IO状态上下文允许我们查看每个单独的操作。

# 使用

函数CreateDBStatistics()创建一个统计对象。

这里有一个例子展示如何把它传递给DB：

```cpp
Options options;
options.statistics = rocksdb::CreateDBStatistics();
```

技术上来说，你可以创建一个统计对象然后传递给多个DB。然后统计对象会包含所有这些DB的数据的聚合结果。注意，在跨多个DB的时候，有些统计值是未定义的，因此他们没有任何意义，比如说"rocksdb.sequence.number"

高级用户可以自行实现自己的统计类。参考最后一章节

# 统计等级和性能损耗

统计带来的损耗通常很小，但是不可以忽视。我们通常会观察到5%~10%的损耗。

统计都使用原子整数来实现（原子递增）。更进一步，统计测量时间间隔需要发起调用来获得当前时间。原子递增和及时函数都会带来损耗，损耗的大小根据平台有所差异。

我们有三个统计等级 kExceptDetailedTimers, kExceptTimeForMutex 和 kAll。

- kAll：收集所有数据，包括计算互斥操作的耗时。如果获取时间在你所在的平台非常昂贵，他会极大减小很多线程的容量，特别是写线程。
- kExceptTimeForMutex：收集所有数据，但是不收集互斥锁内部时间的计数器。rocksdb.db.mutex.wait.micros计数器不会被测量。通过测量这个计数器，我们在DB互斥中调用时间函数。如果时间函数比较慢，这会显著减小写吞吐。
- kExceptDetailedTimers：收集所有信息，除了在互斥锁内的时间**和**压缩的时间。

# 获取统计数据

## 统计类型

有两种类型的统计数据，滴答器和矩形图。

滴答器类型以64bit无符号整形表示。数值不会减小或者重置。滴答统计被用于测量计数器（例如 "rocksdb.block.cache.hit"），累计字节（例如"rocksdb.bytes.written"）或者时间（例如“rocksdb.l0.slowdown.micros”）。

矩阵图类型测量所有操作的统计分布。大多数矩阵图用于DB操作的耗时分布。以“rocksdb.db.get.micros”为例，我们测量每个Get的时间，然后计算他们的分布状态。

## 打印人类可读的字符串

我们可以通过对所有计数器调用ToString()得到人类可读的字符串。

## 在info日志中周期性导出统计日志

统计会自动导出到info日志，时间间隔为options.stats_dump_period_sec。注意，只有一次压缩之后才会有导出。所以如果数据库很长一段时间不做写入，统计数据可能不会导出，不管options.stats_dump_period_sec是多少。

## 代码中访问统计数据

我们还可以通过统计对象直接访问特定的统计数据。滴答器类型列表可以在枚举Tickers里面找到。通过调用statistics.getTickerCount()，我们可以获取一个滴答器的值。类似的，一个矩阵图统计可以通过statistics.histogramData()调用，传入具体的枚举；或者调用statistics.getHistogramString()来获得。

## 时间间隔的统计

所有的统计都是从打开DB就开始累加的。如果你需要根据时间间隔来监控或者报告，你可以周期性地检查数值，然后，通过计算当前值和之前的值，来计算时间间隔值。

# 自定义统计

Statistics是一个抽象类，用户可以实现自己的类，然后传递给options.statistics。如果你需要把RocksDB的统计数据整合到自己的统计系统里面，这就很有用了。当你实现一个自定义的统计的时候，请注意RocksDB对recordTick()和measureTime()的调用量。如果不小心实现，自定义的统计很容易就会变成性能瓶颈。

