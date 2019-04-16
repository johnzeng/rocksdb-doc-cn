如果DB option的`atomic_flush`为true，则RocksDB支持多列族原子落盘。落盘多个列族的执行结果会被写入到MANIFEST文件，使用`all-or-nothing`（要么全部，要么没有）保障（逻辑上的）。使用原子落盘，要么所有，要么没有一个，所有列族的mentalbe都持久化到sst文件，然后添加到数据库。

如果多个列族的数据希望保持一致性，就可以使用这个功能。例如，想想一下，有一个元数据列族`meta_cf`，有一个数据列族`data_cf`，每次我们写入一个新的记录到`data_cf`，我们也会写入到元数据`meta_cf`。`meta_cf`和`data_cf`必须原子化落盘。如果他们其中一个落盘了，但是另一个没有，那么数据就无法保持一致性了。原子化落盘提供一个好的保障。假设在某个特定事件，kv1在`meta_c`的memtable而kv2在`data_cf`的memtable。一次两个列族的原子化落盘后，如果成功，kv1和kv2都会持久化。否则他们都不存在于数据库。

由于原子化落盘会穿越`write_thread`，我们可以保证在批量写的过程中没有落盘发生。

打开、关闭原子罗盘非常简单，使用DB options即可。如果希望以原子落盘方式打开DB，

```cpp
Options options;
... // Set other options
options.atomic_flush = true;
DBOptions db_opts(options);
DB* db = nullptr;
Status s = DB::Open(db_opts, dbname, column_families, &handles, &db);

```

目前，在手动落盘的时候，用户需要负责声明原子落盘的列族。

```cpp
w_opts.disable_wal = true;
db->Put(w_opts, cf_handle1, key1, value1);
db->Put(w_opts, cf_handle2, key2, value2);
FlushOptions flush_opts;
Status s = db->Flush(flush_opts, {cf_handle1, cf_handle2});
```
在RocksDB内部触发的自动落盘，简单起见，我们目前简单地把DB内的所有列族都原子落盘


