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


