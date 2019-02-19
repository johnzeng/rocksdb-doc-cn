# 介绍
在RocksDB3.0，我们增加了Column Families的支持。

RocksDB的每个键值对都与唯一一个列族（column family）结合。如果没有指定Column Family，键值对将会结合到“default” 列族。

列族提供了一种从逻辑上给数据库分片的方法。他的一些有趣的特性包括：

- 支持跨列族原子写。意味着你可以原子执行Write({cf1, key1, value1}, {cf2, key2, value2})。
- 跨列族的一致性视图。
- 允许对不同的列族进行不同的配置
- 即时添加／删除列族。两个操作都是非常快的。

# API

## 向后兼容

尽管我们需要做一些很极端的修改来支持列族，我们还是支持老的API的。你不需要对你的应用做任何改变就可以迁移到RocksDB3.0。所有通过旧的API插入的键值对都会插入到“default”列族。升级之后降级也是同理。如果你从未使用超过一个列族，我们不会改变任何磁盘格式，也就是说你可以放心地回滚到RocksDB2.8。这对我们那些FaceBook里的客户来说，是非常重要的。

## 使用例子

[Column_families_example.cc](https://github.com/facebook/rocksdb/blob/master/examples/column_families_example.cc)

## 参考

Options, ColumnFamilyOptions, DBOptions

在[include/rocksdb/options.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h)中定义，Options结构定义RocksDB的行为和性能。以前，每个option都被定义在单独的Options结构体里。现在，对单个列族的配置，会被定义在ColumnFamilyOptions，然后那些针对整个RocksDB实例的配置会被定义在DBOptions。Options结构体同时继承ColumnFamilyOptions和DBOptions，你仍旧可以用它来设置所有针对只有一个类族的DB实例的配置。

ColumnFamilyHandle

列族通过ColumnFamilyHandle调用和引用。可以把它当成一个打开的文件描述符。在你删除DB指针前，你需要删除所有所有ColumnFamilyHandle。一个有趣的事实：即时一个ColumnFamilyHandle指向一个已经删除的列族，你还是可以继续使用它。数据只有在你将所有存在的ColumnFamilyHandle都删除了，才会被清除。

DB::Open(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr);

当使用读写模式打开一个DB的时候，你需要声明所有已经存在于DB的列族。如果不是，DB::Open会返回 Status::InvalidArgument()，你可以用一个ColumnFamilyDescriptors的vector来声明列族。ColumnFamilyDescriptors是一个只有列族名和ColumnFamilyOptions的结构体。Open会返回一个Status以及一个ColumnFamilyHandle指针的vector，你可以用他们来引用这些列族。删除DB指针前请确保你已经删除了所有ColumnFamilyHandle。


DB::OpenForReadOnly(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr, bool error_if_log_file_exist = false)

行为与DB::Open类似，只不过他用只读模式打开DB。其中一个比较大的差别是，如果用只读模式打开DB，你不需要声明所有列族——你可以只打开一个列族的子集。

DB::ListColumnFamilies(const DBOptions& db_options, const std::string& name, std::vector<std::string>* column_families)

ListColumnFamilies是一个静态方法，会返回当前DB里面存在的列族


DB::CreateColumnFamily(const ColumnFamilyOptions& options, const std::string& column_family_name, ColumnFamilyHandle** handle)

指定一个名字和配置，创建一个列族，然后在参数里面返回一个ColumnFamilyHandle。

DropColumnFamily(ColumnFamilyHandle* column_family)

删除ColumnFamilyHandle指向的列族。注意，实际的数据在客户端调用delete column_family之前 并不会被删除。只要你还有column_family，你就还可以继续使用这个列族。

DB::NewIterators(const ReadOptions& options, const std::vector<ColumnFamilyHandle*>& column_families, std::vector<Iterator*>* iterators)

这是一个新的调用，允许你在DB上创建一个跨列族的一致性视图。

## 批量写

你需要构建一个WriteBatch来实现原子化的批量写操作。所有WriteBatch API现在可以额外携带一个ColumnFamilyHandle指针来声明你希望写到哪个列族。

## 所有其他API调用

所有其他API调用都有了一个新的参数`ColumnFamilyHandle*`，用于声明你想操作的列族。

## 实现

列族的主要实现思想是他们共享一个WAL日志，但是不共享memtable和table文件。通过共享WAL文件，我们实现了酷酷的原子写。通过隔离memtable和table文件，我们可以独立配置每个列族并且快速删除它们。

每当一个单独的列族刷盘，我们创建一个新的WAL文件。所有列族的所有新的写入都会去到新的WAL文件。但是，我们还不能删除旧的WAL，因为他还有一些对其他列族有用的数据。我们只能在所有的列族都把这个WAL里的数据刷盘了，才能删除这个WAL文件。这带来了一些有趣的实现细节以及一些有趣的调优需求。确保你的所有列族都会有规律地刷盘。另外，看一下Options::max_total_wal_size，通过配置他，过期的列族能自动被刷盘。

