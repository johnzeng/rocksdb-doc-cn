Checkpoint在rocksdb中提供了一种 给运行中的数据库在一个独立的文件夹中生成快照的能力。checkpoint可以当成一个特定时间点的快照来使用，可以用只读模式打开，用于查询该时间点的数据，或者以读写模式打开，作为一个可写的快照使用。checkpoint可以用于全量和增量备份。

checkpoint功能使得Rocksdb有能力为给定的Rocksdb数据库在一个特定的文件夹创建一个一致性的快照。如果快照跟原始数据在同一个文件系统，SST文件会以硬链接形式生成，否则，SST文件会被拷贝。MAINFEST文件和CURRENT文件会被拷贝。另外，如果有多个列族，在checkpoint开始和结束的时间段内的日志文件（WAL）会被拷贝，以保证提供一个跨列族的一致性的快照。

生成checkpoint（逻辑意义上的checkpoint）前，一个Checkpoint（cpp代码层面意义上的）对象需要被创建。API如下：

```cpp
Status Create(DB* db, Checkpoint** checkpoint_ptr);
```

给出Checkpoint对象和一个目录，CreateCheckpoint函数会在目标文件夹创建一个数据库的一致性的快照。

```cpp
Status CreateCheckpoint(const std::string& checkpoint_dir);
```

这个目录不应该是已经存在的，他会被这个API创建。目录需要时绝对路径。checkpoint可以被当做一个只读的DB备份，或者可以被当做一个独立的DB实例打开。当以读写模式打开，SST文件会继续以硬链接存在，只有当这些文件被淘汰的时候，这些链接才会被删除。当用户用完这个快照，用户可以删除这个目录，以删除这个快照。

在MyRocks，Checkpoints被用来做在线备份。MyRocks是Mysql使用RocksDB做存储引擎的一个分支（基于Rocksdb的Mysql）。



