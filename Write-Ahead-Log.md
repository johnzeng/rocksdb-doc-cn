# 概述
RocksDB中的每个更新操作都会写到两个地方：1)一个内存数据结构，名为memtable(后面会被刷盘到SST文件) 2)写到磁盘上的WAL日志。在出现崩溃的时候，WAL日志可以用于完整的恢复memtable中的数据，以保证数据库能恢复到原来的状态。在默认配置的情况下，RocksDB通过在每次写操作后对WAL调用fflush来保证一致性。

# WAL的生命周期

我们用一个例子来说明一个WAL的生命周期。一个RocksDB的db实例创建了两个列族："new_cf"和"default"。一旦db被打开，一个新的WAL会在磁盘上被创建，以保证写持久性。

```cpp
DB* db;
std::vector<ColumnFamilyDescriptor> column_families;
column_families.push_back(ColumnFamilyDescriptor(
    kDefaultColumnFamilyName, ColumnFamilyOptions()));
column_families.push_back(ColumnFamilyDescriptor(
    "new_cf", ColumnFamilyOptions()));
std::vector<ColumnFamilyHandle*> handles;
s = DB::Open(DBOptions(), kDBPath, column_families, &handles, &db);
```

往列族中加入一些数据：

```cpp
db->Put(WriteOptions(), handles[1], Slice("key1"), Slice("value1"));
db->Put(WriteOptions(), handles[0], Slice("key2"), Slice("value2"));
db->Put(WriteOptions(), handles[1], Slice("key3"), Slice("value3"));
db->Put(WriteOptions(), handles[0], Slice("key4"), Slice("value4"));
```

这是，WAL需要记录所有的写操作。WAL会保持打开，并不断跟踪后续的写操作，直到他的大小到达`DBOptions::max_total_wal_size`

如果用户决定吧列族"new_cf"的数据落盘，一下的事情会发生

1. new_cf的数据(key1和key3)会被落盘到一个新的SST文件
2. 一个新的WAL会被创建，现在后续的写操作都会写到新的WAL了
3. 旧的WAL不再接受新的写入，但是删除操作会被延后

```cpp
db->Flush(FlushOptions(), handles[1]);
// key5 与 key6 会出现在新的 WAL中
db->Put(WriteOptions(), handles[1], Slice("key5"), Slice("value5"));
db->Put(WriteOptions(), handles[0], Slice("key6"), Slice("value6"));
```

这是，会有两个WAL文件，老的保存有从key1到key4的内容，新的保存key5和key6.因为老的还有线上数据，就是"defalut"列族的，他还不能被删除。只有当用户最后决定把"default"列族的数据落盘，老的WAL才能被归档，然后自动从磁盘上删除。

```cpp
db->Flush(FlushOptions(), handles[0]);
// 老的WAL文件会一步步地被归档，然后删除。
```

总的来说一个WAL文件会在一下时机被创建

1. DB打开的时候
2. 一个列族落盘数据的时候

一个WAL会在他持有的所有列族的数据的最大请求序列号落盘后被删除（或者归档，如果允许归档），换句话说，所有的WAL里的数据都被固定到SST文件。归档WAL会被移到一个独立的位置，然后再从存储设备上清除。实际的删除动作可能会因为拷贝的原因被延后，参考食物日志迭代器章节。

# WAl配置


下面这些配置可以在[options.h](https://github.com/facebook/rocksdb/blob/5.10.fb/include/rocksdb/options.h)中找到

## DBOptions::wal_dir

DBOptions::wal_dir用于设置RocksDB存储WAL文件的目录，这允许用户把WAL和实际数据分开存储

## DBOptions::WAL_ttl_seconds, DBOptions::WAL_size_limit_MB

这两个选项影响WAL文件删除的时间。非0参数表示时间和硬盘空间的阈值，超过这个阀值，会触发删除归档的WAL文件。参考源文件以了解更多内容

## DBOptions::max_total_wal_size

如果希望限制WAL的大小，RocksDB使用DBOptions::max_total_wal_size作为列族落盘的触发器。一旦WAL超过这个大小，RocksDB会开始强制列族落盘，以保证删除最老的WAL文件。这个配置在列族以不固定频率更新的时候非常有用。如果没有大小限制，如果这个WAL中有一些非常低频更新的列族的数据没有落盘，用户可能会需要保存非常老的WAL文件。

## DBOptions::avoid_flush_during_recovery

选项名已经说明了他的用途（回复过程中避免落盘）

## DBOptions::manual_wal_flush

DBOptions::manual_wal_flush决定WAL是每次写操作之后自动flush还是纯人工flush（用户必须调用FlushWAL来触发一个WAL flush）

## DBOptions::wal_filter

通过DBOptions::wal_filter，用户可以提供一个在回复过程中处理WAL文件时被调用的filter对象。注意：ROCKSDB_LITE模式不支持该选项。

## WriteOptions::disableWAL

如果用户依赖于其他写日志方式，或者不担心数据丢失，WriteOptions::disableWAL就非常有用了

# WAL 过滤器

## 事务日志迭代器

事务日志迭代器提供一种方法，用来在RocksDB实例间复制数据。一旦一个WAL因为列族被落盘而被归档，WAL不会马上被删掉。这是为了允许事务日志迭代器可以继续读取WAL文件，再发送给从节点。

## 相关内容

[WAL恢复模式](WAL-Recovery-Modes.md)
[WAL日志格式](Write-Ahead-Log-File-Format.md)



