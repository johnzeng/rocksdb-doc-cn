当落盘或者压缩无法跟上写入速度的时候，RocksDB有额外的系统来降低写速度。如果没有这个系统，如果用户持续写入超出硬件可以处理的数据，数据会：

- 增加空间放大，可能导致磁盘不足
- 增加读放大，显著降级读性能。

主要的思路是降低进来的写入速度到数据库可以处理的级别。然而，有时候数据库可能会对于一个突发写爆发过于敏感，或者低估硬件的处理能力，所以你可能获得非预期的慢速度或者查询超时。

为了确认您的db是否正在一个写失速状态，你可以查看：

- LOG文件，会包含触发写失速的info日志。
- LOG中的[压缩状态]()

失速会因为以下原因被触发：

- **太多的memtable**。当等待落盘的mentable数量大于或等于max_write_buffer_number时，写入会完全停止，以等待落盘完成。另外，如果max_write_buffer_number大于3，并且等待落盘的memtable数量大于或者等于max_write_buffer_number - 1，写入进入失速状态。这是，你会在日志里看到类似于这个的内容：

```
Stopping writes because we have 5 immutable memtables (waiting for flush), max_write_buffer_number is set to 5
```

```
Stalling writes because we have 4 immutable memtables (waiting for flush), max_write_buffer_number is set to 5
```

- **太多level-0的SST文件**。当level 0的SST文件达到level0_slowdown_writes_trigger的时候，写入进入失速状态。当level 0的SST文件达到level0_stop_writes_trigger，写入完全停止，以等待level 0压缩到level 1，以减小level 0的文件数。在这些场景，你会在LOG看到一下info日志

```
Stalling writes because we have 4 level-0 files
```

```
Stopping writes because we have 20 level-0 files
```

- **太多等待压缩的字节**。当预计的等待压缩的字节数达到soft_pending_compaction_bytes，写失速开始。当预计的等待压缩字节数达到hard_pending_compaction_bytes，写入会完全停止，等待压缩。在这种情况，你会在LOG看到以下内容：

```
Stalling writes because of estimated pending compaction bytes 500000000
```

```
Stopping writes because of estimated pending compaction bytes 1000000000
```

不管什么时候，只要失速条件被触发，RocksDB会减小写速率到delayed_write_rate，并且如果预计等待压缩的字节数仍旧增加，可能减小速率到低于delayed_write_rate。一个需要注意的地方时，减速/停止触发器和待压缩字节限制是根据每个列族配置的，但是写失速会应用于整个DB，这意味着如果一个列族触发了写失速，整个DB都会失速。

有几个选项你可以调优以处理写失速。如果你有一些工作场景可以容忍写失速，但是某些不能，你可以设置一些写入到[低优先级写]()以避免哪些延迟敏感的写入失速。

如果写失速是因为落盘被触发，你可以试试：

- 增加max_background_flushes以得到更多的落盘线程
- 增加max_write_buffer_number已获得更小的等待落盘的memtable

如果写失速是因为太多level 0文件或者过多等待压缩的数据触发，压缩不够快，追不上写入了。注意任何减小写放大会减小压缩的时候需要写的字节数，因此可以加快压缩。可以是的选项：

- 增加max_background_compactions来获得更多压缩线程
- 增加write_buffer_size已获得更大的memtable，以减小写放大
- 增加min_write_buffer_number_to_merge。

你也可以设置停止/减速开关以及等待压缩字节数限制到大数字，以避免触发写失速。另外，如果你正在加载一大批数据，看一下[FAQ](RocksDB-FAQ.md)中关于"如何最快速加载数据到RocksDB"的内容。

