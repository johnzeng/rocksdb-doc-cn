这个功能会删除一个key的最后一个版本的值，但是旧的版本的值会不会重新出现，**不确定**

# 基本使用

SingleDelete是一个新的数据库操作。与传统的Delete操作不同，被删除的项，会在压缩的时候与数值一起被移除。因此，与Delete类似，SingleDelete删除一个key，但是有一个前提，就是这个key存在且没有被覆盖过。成功返回OK，否则返回非OK。如果key不存在，不会报错。如果key被覆盖过（多次调用Put），那么对一个key调用SingleDelete会导致未定义行为。只有当一个key在上次调用过SingleDelete之后只被Put过一次，SingleDelete才能正确执行。这个功能目前还是为了处理某些非常特别的工作的实验性质的。下面一段代码展示了如何使用SingleDelete：

```cpp
std::string value;
  rocksdb::Status s;
  db->Put(rocksdb::WriteOptions(), "foo", "bar1");
  db->SingleDelete(rocksdb::WriteOptions(), "foo");
  s = db->Get(rocksdb::ReadOptions(), "foo", &value); // s.IsNotFound()==true
  db->Put(rocksdb::WriteOptions(), "foo", "bar2");
  db->Put(rocksdb::WriteOptions(), "foo", "bar3");
  db->SingleDelete(rocksdb::ReadOptions(), "foo", &value); // Undefined result
SingleDelete API is also available in WriteBatch. Actually, DB::SingleDelete() is implemented by creating a WriteBatch with only one operation, SingleDelete, in this batch. The following code snippet shows the basic usage of WriteBatch::SingleDelete():
  rocksdb::WriteBatch batch;
  batch.Put(key1, value);
  batch.SingleDelete(key1);
  s = db->Write(rocksdb::WriteOptions(), &batch);

```

## 注意

- 调用者必须却表SingleDelete只用在那些没有被Delete删除，或者使用Merge写入过的key。混合使用SingleDelete，Delete和Merge会导致未定义行为（其他的key不会受影响）。
- SingleDelete与cuckoo哈希表不兼容，如果你把options.memtable_factory设置为[NewHashCuckooRepFactory](https://github.com/facebook/rocksdb/blob/522de4f59e6314698286cf29d8a325a284d81778/include/rocksdb/memtablerep.h#L325),那么你就无法使用SingleDelete
- 不允许连续的SingleDelete
- 考虑设置write_options.sync为true



