DeleteRange这个操作，被设计出来替换下面这种用户需要删除一整段key的场景。

```cpp
...
Slice start, end;
// set start and end
auto it = db->NewIterator(ReadOptions());

for (it->Seek(start); cmp->Compare(it->key(), end) < 0; it->Next()) {
  db->Delete(WriteOptions(), it->key());
}
...
```

这种场景需要执行一个范围扫描，这就导致无法做到原子化操作，并且无法满足性能敏感的写场景。为了解决这个问题，RocksDB提供了一个院子操作来解决这个任务：

```cpp
...
Slice start, end;
// set start and end
db->DeleteRange(WriteOptions(), start, end);
...
```

底层，他会创建一个范围墓碑，表现上就是一个kv对，会显著提升写速度。范围扫描的读性能则与 扫描-删除 模式向兼容（更加详细的性能分析，参考[DeleteRange Blog](https://rocksdb.org/blog/2018/11/21/delete-range.html)）。

