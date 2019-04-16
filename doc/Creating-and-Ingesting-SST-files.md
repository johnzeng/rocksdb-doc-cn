Rocksdb向用户提供了一系列API用于创建及导入SST文件。在你需要快速读取数据但是数据的生成是离线的时候，这非常有用。

## 创建SST文件

rocksdb::SstFileWriter可以用于创建SST文件。创建了一个rocksdb::SstFileWriter 对象之后，你可以打开一个文件，插入几行数据，然后结束。

这里有一个例子，展示了如何创建SST文件/home/usr/file1.sst

```cpp
Options options;
SstFileWriter sst_file_writer(EnvOptions(), options);
// Path to where we will write the SST file
std::string file_path = "/home/usr/file1.sst";

// Open the file for writing
Status s = sst_file_writer.Open(file_path);
if (!s.ok()) {
    printf("Error while opening file %s, Error: %s\n", file_path.c_str(),
           s.ToString().c_str());
    return 1;
}

// Insert rows into the SST file, note that inserted keys must be 
// strictly increasing (based on options.comparator)
for (...) {
  s = sst_file_writer.Put(key, value);
  if (!s.ok()) {
    printf("Error while adding Key: %s, Error: %s\n", key.c_str(),
           s.ToString().c_str());
    return 1;
  }
}

// Close the file
s = sst_file_writer.Finish();
if (!s.ok()) {
    printf("Error while finishing file %s, Error: %s\n", file_path.c_str(),
           s.ToString().c_str());
    return 1;
}
return 0;

```

现在我们有了一个在/home/usr/file1.sst的SST文件了。

注意：

- 传给SstFileWriter的Options会被用于指定表类型，压缩选项等，用于创建sst文件。
- 传入SstFileWriter的Comparator必须与之后导入这个SST文件的DB的Comparator绝对一致。
- 行必须严格按照增序插入

参考[nclude/rocksdb/sst_file_writer.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/sst_file_writer.h) 了解更多的内容

## 导入SST文件

导入SST文件非常简单，你所需要做的只是调用DB::IngestExternalFile()然后吧文件地址以std::string的vector传入就行了

```cpp
IngestExternalFileOptions ifo;
// Ingest the 2 passed SST files into the DB
Status s = db_->IngestExternalFile({"/home/usr/file1.sst", "/home/usr/file2.sst"}, ifo);
if (!s.ok()) {
  printf("Error while adding file %s and %s, Error %s\n",
         file_path1.c_str(), file_path2.c_str(), s.ToString().c_str());
  return 1;
}
```

参考[include/rocksdb/db.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/db.h)了解更多内容

## 导入一个文件的时候发生了什么？

### 当你调用DB::IngestExternalFile()我们会：
- 把文件拷贝，或者链接到DB的目录
- 阻塞DB的写入（而不是跳过），因为我们必须保证db状态的一致性，所以我们必须确保，我们能给即将导入的文件里的所有key都安全地分配正确的序列号。
- 如果文件的key覆盖了memtable的键范围，把memtable刷盘
- 把文件安排到LSM树的最好的层
- 给文件赋值一个全局序列号
- 重新启动DB的写

### 我们选择LSM树中满足以下条件的最低的一个层
- 文件可以安排在这个层
- 这个文件的key的范围不会覆盖上面任何一层的数据
- 这个文件的key的范围不会覆盖当前层正在进行压缩

### 全局序列号

通过SstFileWriter创建的文件的元信息块里有一个特别的名为全局序号的字段，当这个字段第一次被使用，这个文件里的所有key都认为自己拥有这个序列号。当我们导入一个文件，我们给文件里的所有key都分配一个序列号。RocksDB5.16之前，RocksDB总是用一个随机写来更新这个元数据块里的全局序列号字段。从RocksDB5.16之后，RocksDB允许用户选择是否通过IngestExternalFileOptions::write_global_seqno更新这个字段。如果这个字段在导入的过程中没有被更新，那么RocksDB在读取文件的时候使用MANIFEST的信息以及表属性来推断全局序列号。如果底层的文件系统不支持随机写，这个路径就非常有效了。考虑到向后兼容，可以把这个选项设置为true，这样RocksDB 5.16或者更新的版本生成的SST文件就可被5.15或者更旧的Rocksdb也可以打开这个文件了。

## 下层导入

从5.5开始，IngestExternalFile加载一个外部SST文件列表的时候，支持下层导入，意思是，如果ingest_behind为true，那么重复的key会被跳过。在这种模式下，我们总是导入到最底层。文件中重复的key会被跳过，而不是覆盖已经存在的key。

### 使用场景

回读部分历史数据，而不覆盖最新的数据。这个选项只有在DB一开始运行的时候增加了allow_ingest_behind=true选项才可以使用。所有的文件都会被导入到最底层，seqno=0




