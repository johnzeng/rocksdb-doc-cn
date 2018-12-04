#RocksDB选项文件

从RocksDB4.3开始，我们加了一系列功能来简化RocksDB的设置。

- 每次成功调用DB::Open()，SetOptions()，以及CreateColumnFamily和DropColumnFamily被调用的时候，RocksDB的数据库都会自动将当前的配置持久化到一个文件里。
- [LoadLatestOptions() / LoadOptionsFromFile()](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/utilities/options_util.h#L20-L58) ：用于从一个选项文件构造RocksDB选项。
- [CheckOptionsCompatibility](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/utilities/options_util.h#L64-L77) :一个用于检查两个RocksDB选项的兼容性。

通过上面这些选项文件的支持，开发者再也不用维护以前的RocksDB数据库对象的配置集。另外，如果需要修改选项，CheckOptionsCompatibility可以保证新的配置集可以在不损坏已有数据的情况下打开同一个RocksDB数据库。

## 示例

这里有一个可以运行的示例，用于展示新的功能是如何让管理RocksDB选项更简单的。一个更加完整的例子可以在[examples/options_file_example.cc](https://github.com/facebook/rocksdb/blob/master/examples/options_file_example.cc) 里找到。

假设我们打开一个RocksDB数据库，然后在运行过程中创建一个新的列族，然后关闭数据库：

```cpp
s = DB::Open(rocksdb_options, path_to_db, &db);
...
// 创建列族，然后rocksdb会保存这些选项。
ColumnFamilyHandle* cf;
s = db->CreateColumnFamily(ColumnFamilyOptions(), "new_cf", &cf);
...
// 关闭 DB
delete cf;
delete db;
```

从RocksDB4.3之后，每个RocksDB实例都会自动将最新的配置存储在一个配置文件，下次打开数据库的时候，我们可以利用这个配置文件来构造新的选项对象。这跟RocksDB 4.2以及更老的版本不同，我们再也不用为了打开一个数据库而记住所有列族的选项了。我们看看要如何做。

首先，我们使用LoadLatestOptions加载目标DB最新的设置：

```cpp
DBOptions loaded_db_opt;
std::vector<ColumnFamilyDescriptor> loaded_cf_descs;
LoadLatestOptions(path_to_db, Env::Default(), &loaded_db_opt,
                  &loaded_cf_descs);

```

## 不支持的选项

由于c++没有反射机制，以下需要使用用户定义的函数以及指针类型的配置项，只能被初始化为默认值。详细信息可以参考rocksdb/utilities/options_util.h：

- env
- memtable_factory
- compaction_filter_factory
- prefix_extractor
- comparator
- merge_operator
- compaction_filter
- BlockBasedTableOptions里的缓存
- table_factory， 除了 BlockBasedTableFactory

对于那些不支持的用户定义函数，开发者需要手动指定他们。在这个例子，我们初始化BlockBasedTableOptions里的缓存和CompactionFilter：

```cpp
for (size_t i = 0; i < loaded_cf_descs.size(); ++i) {
  auto* loaded_bbt_opt = reinterpret_cast<BlockBasedTableOptions*>(
      loaded_cf_descs[0].options.table_factory->GetOptions());
  loaded_bbt_opt->block_cache = cache;
}

loaded_cf_descs[0].options.compaction_filter = new MyCompactionFilter();

```

现在我们执行安全性检查，确保新的选项可以安全打开目标数据库：

```cpp
Status s = CheckOptionsCompatibility(
    kDBPath, Env::Default(), db_options, loaded_cf_descs);

```

如果返回的值是OK，我们就可以继续使用加载起来的选项来打开目标数据库了：

```cpp
s = DB::Open(loaded_db_opt, kDBPath, loaded_cf_descs, &handles, &db);
```

## RocksDB选项文件格式

RocksDB选项文件是一个 [INI文件格式](https://en.wikipedia.org/wiki/INI_file) 的text文件。每个RocksDB配置文件都一个版本号段，一个DBOptions段以及对每个列族，有一个CFOptions和TableOptions段。以下是一个示例的RocksDB选项文件。一个完整的示例可以在 [examples/rocksdb_option_file_example.ini](https://github.com/facebook/rocksdb/blob/master/examples/rocksdb_option_file_example.ini)找到

```ini
[Version]
  rocksdb_version=4.3.0
  options_file_version=1.1
[DBOptions]
  stats_dump_period_sec=600
  max_manifest_file_size=18446744073709551615
  bytes_per_sync=8388608
  delayed_write_rate=2097152
  WAL_ttl_seconds=0
  ...
[CFOptions "default"]
  compaction_style=kCompactionStyleLevel
  compaction_filter=nullptr
  num_levels=6
  table_factory=BlockBasedTable
  comparator=leveldb.BytewiseComparator
  compression_per_level=kNoCompression:kNoCompression:kNoCompression:kSnappyCompression:kSnappyCompression:kSnappyCompression
  ...
[TableOptions/BlockBasedTable "default"]
  format_version=2
  whole_key_filtering=true
  skip_table_builder_flush=false
  no_block_cache=false
  checksum=kCRC32c
  filter_policy=rocksdb.BuiltinBloomFilter
  ....
```


