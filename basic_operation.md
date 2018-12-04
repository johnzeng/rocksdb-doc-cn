#基础操作

Rocksdb库提供一个持久化的键值存储。健和值都可以是任意二进制数组。所有的键按爪一个用户定义的比较函数排列。

## 打开一个数据库

一个rocksdb数据库会有一个与文件系统目录关联的名字。所有与该数据库有关的内容都会存储在那个目录里。下面的例子将展现如何打开一个数据库，有必要的时候创建他：

```cpp
  #include <cassert>
  #include "rocksdb/db.h"

  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  assert(status.ok());
  ...
```

如果你希望在数据库存在的时候返回错误，在调用rocksdb::DB::Open调用前，增加下面这行。

options.error_if_exists = true;

如果你正在从leveldb迁移到rocksdb，你可以使用rocksdb::LevelDBOptions把你的leveldb::Options对象转成rocksdb::Options对象，他有与leveldb::Options一样的功能。

```cpp
#include "rocksdb/utilities/leveldb_options.h"

  rocksdb::LevelDBOptions leveldb_options;
  leveldb_options.option1 = value1;
  leveldb_options.option2 = value2;
  ...
  rocksdb::Options options = rocksdb::ConvertOptions(leveldb_options);

```

## RocksDB选项

如上所示，用户可以选择总是在代码里明确设置选项的内容。或者，你可以通过一个字符串到字符串的map，或者一个选项字符串来设置。参考[选项字符串和选项Map]()

有一些选项可以在DB运行过程中动态修改。例如：

```cpp
rocksdb::Status s;
s = db->SetOptions({{"write_buffer_size", "131072"}});
assert(s.ok());
s = db->SetDBOptions({{"max_background_flushes", "2"}});
assert(s.ok());
```

RocksDB自动将当前数据库使用的配置保存到数据库目录下的OPTIONS-xxx文件。用户可以选择在数据库重启之后从配置文件导出选项，以此来保存配置。参考 [RocksDB配置文件]()

##状态(Status)

你可能已经注意到上面的rocksdb::Status类型。这个类型的值是大部分rocksdb的返回值，他有时会返回一个错误。你可以检测这个结果是不是ok，然后打印相关的错误信息：

```cpp
 rocksdb::Status s = ...;
   if (!s.ok()) cerr << s.ToString() << end

```

## 关闭一个数据库

当你使用完这个数据库，只需要删除这个数据库对象即可：

```cpp
  ... 按上面的描述打开数据库 ...
  ... 对这个数据库做一些操作 ...
  delete db;
```

## 读与写

数据库提供Put,Delete以及Get方法用于修改/查询数据库。例如，下面的代码将key1的值，移到key2。

```cpp
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) s = db->Put(rocksdb::WriteOptions(), key2, value);
  if (s.ok()) s = db->Delete(rocksdb::WriteOptions(), key1);

```

目前，值的大小必须小于4GB。RocksDB还允许使用 [单删除]()，这个在特定场景很有用。

每个Get请求至少需要使用一次从源到目的字符串的值memcpy拷贝。如果源在块缓存，你可以使用一个PinnableSlice来避免额外的拷贝。

```cpp
 PinnableSlice pinnable_val;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &pinnable_val);

```

源数据会在pinnable_val被销毁或者::Reset被调用的时候释放。参考[这里]()

##原子化更新

主意，如果进程在key2的Put调用之后，在删除key1之前，崩溃了，那么相同的值会被保存到多个键下面。这种问题可以通过WriteBatch类来原子地写入一批更新：

```cpp
  #include "rocksdb/write_batch.h"
  ...
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) {
    rocksdb::WriteBatch batch;
    batch.Delete(key1);
    batch.Put(key2, value);
    s = db->Write(rocksdb::WriteOptions(), &batch);
  }

```

WriteBatch保存一个数据库编辑序列，这些批处理的修改会被按顺序应用于数据库。主意我们在Put之前使用Delete，所以如果key1和key2相同，我们不会错误地将这个值删掉。

除了他原子化的有点，WriteBatch还可以通过将大量的单独修改合并到一个批处理，以此加速批量更新。

## 批量写

rocksdb的写请求默认都是同步的：他在进程把写请求压入操作系统后返回。从操作系统的内存写入之下的持续存储介质的过程是异步的。可以对某个特定的写请求使用sync标识位，使之在数据完全被写入持久化存储介质之前都不会反回。（在Posix系统，可以在写操作返回前，调用fsync或者fdatasync或者msync。）

```cpp
  rocksdb::WriteOptions write_options;
  write_options.sync = true;
  db->Put(write_options, ...);
```

## 异步写

通常异步写会比同步写快上千倍。异步写的缺点是机器崩溃的时候可能会造成最后一个更新操作丢失。主意，如果只是写操作的进程崩溃，即使sync标示没有设置为真，也不会造成数据丢失，在他的写操作返回成功前，更新操作会从进程的内存压入操作系统。

异步写通常可以被安全地使用。例如，在加载一大批数据的时候，你可以通过重启批量加载操作来处理由于崩溃导致的数据丢失。一个混合方案是，你可以每隔几个写操作，就使用一次同步写，当崩溃发生的时候，批量导入可以从上一次运行的最后一次同步写那里继续。（同步写可以更新一个标记，用于纪录下次重启的时候应该从哪里开始）。

WriteBatch提供另一个异步写的方案。可以把许多更新打包在一个WriteBatch里面，然后一次性用一个同步写导入数据库。（write_options.sync被设置为真）。同步写的开销会被该批量写的所有写请求均匀分摊。

我们还提供一个办法，在有必要的时候，可以彻底关闭WAL。如果你把write_option.disableWAL设置为true，写操作完全不会写日志，并且在进程崩溃的时候出现数据丢失。

## 同步

一个数据库可能同时只能被一个进程打开。RocksDB的实现会从操作系统那里申请一个锁来阻止错误的写操作。在单进程里面，同一个rocksdb::DB对象可以被多个同步线程共享。举个例子，不同的线程可以同时对同一个数据库调用写操作，迭代器遍历操作或者Get操作，而且不用使用额外的同步设定。（rocksdb的实现会自动进行同步）。然而其他对象（比如迭代器，WriteBatch）需要额外的同步。如果两个线程共享这些对象，他们必须使用他们自己的锁协议保证访问的同步。更多的细节会在公共头文件给出。

## 合并操作符
合并操作符为 读－修改－写 操作提供高效的支持。更多的接口和实现参考：

- [合并操作符]()
- [合并操作符开发]()

## 迭代器

下面的例子展示如何打印一个数据库里的所有键值对（key，value）。

```cpp
  rocksdb::Iterator* it = db->NewIterator(rocksdb::ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    cout << it->key().ToString() << ": " << it->value().ToString() << endl;
  }
  assert(it->status().ok()); // Check for any errors found during the scan
  delete it;
The following variation shows how to process just the keys in the range [start, limit):
  for (it->Seek(start);
       it->Valid() && it->key().ToString() < limit;
       it->Next()) {
    ...
  }
  assert(it->status().ok()); // Check for any errors found during the scan

```

你还可以通过反向的顺序处理这些项目。（警告：反响迭代器会比正向迭代器慢一些）

```cpp
for (it->SeekToLast(); it->Valid(); it->Prev()) {
    ...
  }
  assert(it->status().ok()); // Check for any errors found during the scan

```

这里有一个例子展示如何以逆序处理一个范围(limit,start]的键

```cpp
  for (it->SeekForPrev(start);
       it->Valid() && it->key().ToString() > limit;
       it->Prev()) {
    ...
  }

	assert(it->status().ok()); // Check for any errors found during the scan

```

参考 [SeekForPrev]()

对于错误处理的解释，不同的迭代选项和最佳实践，参考 [迭代器]()

了解更多的实现细节，参考： [迭代器的实现]()

## 快照

快照在整个kv存储之上提供一个一致的，制度的视图。 ReadOptions::snapshot不是NULL的时候意味着这个读操作应该在某个特定的数据库状态版本进行。

如果 ReadOptions::snapshot是NULL，读操作隐式地认为使用当前数据库状态进行读操作。

快照通过DB::GetSnapshot方法获得：

```cpp
  rocksdb::ReadOptions options;
  options.snapshot = db->GetSnapshot();
  ... apply some updates to db ...
  rocksdb::Iterator* iter = db->NewIterator(options);
  ... read using iter to view the state when the snapshot was created ...
  delete iter;
  db->ReleaseSnapshot(options.snapshot);

```

注意，当一个快照不再需要了，他应该通过DB::ReleaseSnapshot接口释放。这样才能让数据库的实现摆脱那些只为了快照保留下来的数据。

## Slice
上面调用的it->key()和it->value()的返回值类型是rocksdb::Slice类型。Slice是一个简单的结构体，他有一个长度字段和一个指针指向一个外部的字节数组。相比返回一个std::string类向，返回一个Slice是一个开销更低的选项，因为我们不必另外去拷贝那些可能会很大的键值对。另外，rocsdb的方法不会返回以null结束的c风格字符串，因为rocksdb的键值都是允许使用'\0'字符的。

c++字符串和null结束的c风格字符串，都可以简单地转换为slice：

```cpp
   rocksdb::Slice s1 = "hello";

   std::string str("world");
   rocksdb::Slice s2 = str;

```

一个Slice可以简单地转换会c++字符串：

```cpp
   std::string str = s1.ToString();
   assert(str == std::string("hello"));
```
使用slice的时候要小心，因为需要由调用者来保证外部的字节数组在Slice使用期间存活。比如，下面这个代码就是有bug的：

```cpp
   rocksdb::Slice slice;
   if (...) {
     std::string str = ...;
     slice = str;
   }
   Use(slice);

```

当if声明结束的时候，str会被析构，然后slice存储的数据就消失了。

## 事务
RocksDB现在支持多操作事务。参考 [事务]()

## 比较器
前面的例子使用默认的排序函数对键值进行排序，也就是使用字典顺序排列。你也可以在打开数据库的时候使用自定义的比较器。例如，假如每个数据库的键值都是两个数字，然后我们应该按第一个数字排序，如果第一个数字相同，按照第二个数字排序。首先，定义一个合适的rocksdb::Comparator子类，实现下面的规则：

```cpp
  class TwoPartComparator : public rocksdb::Comparator {
   public:
    // Three-way comparison function:
    // if a < b: negative result
    // if a > b: positive result
    // else: zero result
    int Compare(const rocksdb::Slice& a, const rocksdb::Slice& b) const {
      int a1, a2, b1, b2;
      ParseKey(a, &a1, &a2);
      ParseKey(b, &b1, &b2);
      if (a1 < b1) return -1;
      if (a1 > b1) return +1;
      if (a2 < b2) return -1;
      if (a2 > b2) return +1;
      return 0;
    }

    // Ignore the following methods for now:
    const char* Name() const { return "TwoPartComparator"; }
    void FindShortestSeparator(std::string*, const rocksdb::Slice&) const { }
    void FindShortSuccessor(std::string*) const { }
  };

```

现在，使用自定义的比较器打开数据库

```cpp
  TwoPartComparator cmp;
  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  options.comparator = &cmp;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  ...

```

## 列族

列族提供一种逻辑上给数据库分区的方法。用户可以提供一个原子的，跨列族的多键写操作以及通过一个一致的视图读他们。

## 批量加载

你可以通过[创建并导入SST文件]()来将导入大量的数据直接，批量导入数据库，并且将线上流量做到最小的影响。

## 备份以及检查点

[备份]()允许用户创建周期性的增量备份到远程文件系统（想想HDFS和S3），然后从他们中的任意一个恢复。

[检查点]()提供一种能力，为线上的RocksDB生成一个快照到一个独立的目录。文件通过硬链接（如果可以的话），而不是拷贝生成，所以这个是相对轻量的一个操作

##I/O

默认RocksDB的IO会使用操作系统的页缓存。通过设置 [速度限制器]()可以限制rocksdb的写文件操作速度，给读IO留下空间。

用户也可以选择直接跳过页缓存，使用 [直接IO]()

参考 [IO]()

##向后兼容

数据库创建的时候，比较器的 Name方法返回的结果会被追加到数据库，然后每次重新打开的时候都会被检查。如果名称修改了，rocksdb::DB::Open调用会失败。所以，当且仅当新的键组织和比较器跟当前已经存在的数据库有冲突，并且可以丢弃旧的所有数据库的数据的时候，才修改这个名称。

当然，你也可以有计划的一点点地修改你的键格式。例如，你可以在每个键的末尾存一个版本号（对于多数用户，一个byte应该足够了）。当你希望切换到新的键组织的时候，（例如，增加一个可选的第三部分键处理给以前TwoPartComparator处理过的键），(a)保留比较器的名字(b)新的键增加版本号(c)修改比较器，按照版本号决定如何解析他们。

## Memtable和table工厂

默认，我们用skiplist memtable保存内存的数据，然后用 [RocsDB表格式]() 描述的表格式保存硬盘的数据。

由于RocksDB的其中一个目标是让系统里面的每个部分都可以是简单的插件，我们支持对memtable和表格式使用不同的实现。你也可以使用自己的memtable工厂，只要设置Options::memtable_factory 和Options::table_factory即可。对于可用的memtable工厂类，请参考rocksdb/memtablerep.h，表格式参考 rocksdb/table.h。这些功能都在开发中，请注意某些API的变化，可能会影响到你的应用走向。

更多关于memtable的内容参考 [这里]() 和 [这里]()

## 性能

从 [设置配置]() 开始。更多关于RocksDB的性能信息，参考旁边的Performance章节

## 块大小

rocksdb将一组连续的键打包到一个块，这个块就是跟持久存储的交换单位。默认的块大小是接近4096byte（压缩前）。一些经常需要做区间扫描的程序，可能希望增加这个大小。对于一些大量点查询的应用，如果确实能看到性能提升，会希望使用一个更小的值。使用一个小于1kB的块大小一般不会有特别多的好处，使用大于几个MB同理。主意，压缩对于越大的块效率越高。使用Options::block_size修改块大小

## 写缓冲区

Options::write_buffer_size选项指定那些还没排序并存入磁盘文件的数据在内存里面可以占用的空间大小。更大的值一般带来更好的性能，特别是在大批量导入数据的时候。同时最多有max_write_buffer_number个写缓冲区在内存，你可以通过这个参数控制内存消耗。同时，更大的写缓冲区意味着数据库重启的时候需要更长的恢复时间。

相关选项是Options::max_write_buffer_number，用于控制内存中写缓冲区的数量。默认为2，这样，当一个写缓冲区正在被落盘的时候，一个新的可以继续服务新的写请求。落盘操作会在一个[线程池]()中进行。

选项Options::min_write_buffer_number_to_merge声明在落盘前，需要合并的写缓冲区的最小数量。如果设置为1，那么所有的写缓冲区都与L0的一个文件对应，这会增加写放大，因为每次读都要检查所有的这些文件。同时，如果每个写缓冲区都有一些重复的键，内存中的合并操作可以减少落盘数据的量。默认值为1。

## 压缩

每个块都会在写入持久化存储前进行压缩。压缩是默认开启的，因为默认的压缩算法非常快，对于不可压缩的数据，则被自动关闭。在非常罕见的情况下，应用会希望彻底关闭压缩，除非你的压测显示这确实带来了好处，否则不要这么做：

```cpp
  rocksdb::Options options;
  options.compression = rocksdb::kNoCompression;
  ... rocksdb::DB::Open(options, name, ...) ....

```

同时，我们还提供 [字典压缩]()

## 缓存

数据库的数据会被存储到文件系统的一系列文件里，每个文件存储一部分压缩好的块。如果 options.block_cache为非NULL，他会被用于缓存最常用的解压后的块的内容。我们使用操作系统来缓存原始的，压缩的数据。文件系统缓存扮演着压缩数据缓存的角色。

```cpp
  #include "rocksdb/cache.h"
  rocksdb::BlockBasedTableOptions table_options;
  table_options.block_cache = rocksdb::NewLRUCache(100 * 1048576); // 100MB uncompressed cache

  rocksdb::Options options;
  options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));
  rocksdb::DB* db;
  rocksdb::DB::Open(options, name, &db);
  ... use the db ...
  delete db

```

执行批量读取的时候，应用会希望关闭缓存，这样批量读区就不会污染已经缓存的数据。一个针对迭代器的选项可以做到这个：

```cpp
  rocksdb::ReadOptions options;
  options.fill_cache = false;
  rocksdb::Iterator* it = db->NewIterator(options);
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    ...
  }
```

你也可以通过把options.no_block_cache设置为true来关闭块缓存。

参考 [块缓存]()

## 键分布

注意，缓存与磁盘交换数据的单位是块。连续的键（根据数据库的排序）通常被放在同一个块。所以应用也可以通过把常常一起使用的键放在一起，然后把另一些不常用的放在另一个命名空间，以此提高性能。

例如，加入我们机遇rocksdb开发一个简单的文件系统。每个节点的类型可能这样存储：

```
   filename -> permission-bits, length, list of file_block_ids
   file_block_id -> data

```

我们可能希望给filename这个键使用一个前缀（例如'/'），然后给file_block_id使用另一个前缀（例如'0'），这样，扫描元数据的时候就不用关心大量的文件内容信息了。

## 过滤器
由于Rocksdb在硬盘的数据组织方式，一个Get请求可能会导致多个磁盘读请求。这个时候，FilterPolicy机制就可以用来非常可观地减少磁盘读。

```cpp
   rocksdb::Options options;
   rocksdb::BlockBasedTableOptions bbto;
   bbto.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10));
   options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(bbto));
   rocksdb::DB* db;
   rocksdb::DB::Open(options, "/tmp/testdb", &db);
   ... use the database ...
   delete db;
   delete options.filter_policy;

```

这段代码需要对数据库使用一个基于 [BloomFilter]() 的过滤策略。基于Bloom Filter的过滤器会在内存保留一部分键的内容（这里是10bit，因为我们传递给NewBloomFilter的参数就是这个）。这个过滤器可以在Get请求的时候减少大概100倍不必要的磁盘读。增大每个键的bit数会导致更多的削减，但是回来带更多的内存使用。我们推荐那些数据没法全部存储在内存，但是有大量随机读的应用使用过滤策略。

如果你使用自定义的比较器，你需要保证你的过滤策略跟你的比较器是兼容的。例如，如果有一个比较器，比较键的时候，不关心末尾的空格。那么，NewBloomFilter就不能给这种比较器使用了。作为替代，应用应该提供一个自定义的过滤策略，忽略掉这些末尾的空格。

比如说：

```
  class CustomFilterPolicy : public rocksdb::FilterPolicy {
   private:
    FilterPolicy* builtin_policy_;
   public:
    CustomFilterPolicy() : builtin_policy_(NewBloomFilter(10, false)) { }
    ~CustomFilterPolicy() { delete builtin_policy_; }

    const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

    void CreateFilter(const Slice* keys, int n, std::string* dst) const {
      // Use builtin bloom filter code after removing trailing spaces
      std::vector<Slice> trimmed(n);
      for (int i = 0; i < n; i++) {
        trimmed[i] = RemoveTrailingSpaces(keys[i]);
      }
      return builtin_policy_->CreateFilter(&trimmed[i], n, dst);
    }

    bool KeyMayMatch(const Slice& key, const Slice& filter) const {
      // Use builtin bloom filter code after removing trailing spaces
      return builtin_policy_->KeyMayMatch(RemoveTrailingSpaces(key), filter);
    }
  };
```
上面的应用提供一个不是使用bloom filter的过滤策略，而是使用其他的策略来提取一个键值集合的值。参考rocksdb/filter_policy.h

## 校验和

rocksdb会给所有存储在文件系统的数据添加校验和。有两个独立的方法控制校验和的校验强度。

- ReadOptions::verify_checksums 强制要求某个读请求对所有的从硬盘读的数据都要检查校验和。这个是默认开的。
- Options::paranoid_checks 如果在打开数据库的时候被设置为了true，那么数据库的实现就会在他检测到校验和错误的时候返回一个错误。根据数据库具体出错的部位，这个错误可能在数据库打开的时候就反回，也可能在后续的其它操作反回。默认paranoid_checks为false，这样即使部分持久化数据出错，DB也还可以继续运作。

如果db崩溃了（比如paranoid_checks为true时打开失败了），rocksdb::RepairDB方法**可能**可以用来尽可能地恢复数据。

## 压缩

RocksDB会不停地重写已经存在的数据文件。这是为了丢掉已经过期的数据，并且保证数据结构对读友好。

与压缩有关的内容已经移动到 [压缩]()章节，用户操作RocksDB前不需要知道哪部压缩过程。

## 估算大小

GetApproximateSizes方法可以用于获得一个或多个键占用的文件系统的空间的大概值。

```cpp
   rocksdb::Range ranges[2];
   ranges[0] = rocksdb::Range("a", "c");
   ranges[1] = rocksdb::Range("x", "z");
   uint64_t sizes[2];
   rocksdb::Status s = db->GetApproximateSizes(ranges, 2, sizes);
```

上面的代码会把sizes[0]填写成键空间[a...c)占用的文件系统大概的大小，然后sizes[1]则会填写成键空间[x..z)的。

## 环境

所有rocksdb实现的，发起的文件操作（以及其他系统调用），都是通过一个rocksdb::Env对象来实现的。一些熟悉原理的客户可能希望使用自己的Env实现来获得更好的控制。例如，一个应用可能会人为制造一些文件IO的延迟，来限制rocksdb对该系统上的其它活动的影响。

```cpp
  class SlowEnv : public rocksdb::Env {
    .. implementation of the Env interface ...
  };

  SlowEnv env;
  rocksdb::Options options;
  options.env = &env;
  Status s = rocksdb::DB::Open(options, ...);
```

## 移植

rocksdb可以通过实现rocksdb/port/port.h导出的类型/方法/函数，来移植到一个新的平台。参考rocksdb/port/port_example.h

另外，移植到一个新的平台可能需要实现一个新的rocksdb::Env。参考 rocksdb/util/env_posix.h

## 可管理性

为了更有效地调优你的应用， 如果能获得一些统计数据，总是很有用的。你可以通过设置Options::table_properties_collectors或者Options::statistics来收集统计信息。其他信息，可以参考 rocksdb/table_properties.h和 rocksdb/statistics.h。这个不会给你的系统带来太多的负担，我们推荐你把他们导出到其它监控系统。参考 [统计]()， 你也可以使用[上下文及IO状态剖析]()对单一请求进行剖析。用户还可以对一些内部事件注册 [事件监听器]() 。

## 清理WAL文件

默认，旧的WAL文件会在他们已经掉出范围并且没人需要的时候被自动删除。我们也提供选项允许用户归档日志并且做懒删除，既可以是TTL风格，也可以根据空间限制。

这些选项就是Options::WAL_ttl_seconds和Options::WAL_size_limit_MB。这里展示怎么使用它们

- 如果都是0，logs会尽可能快的被删除，而且不会被归档。
- 如果WAL_ttl_seconds是0，但是WAL_size_limit_MB不是0，WAL文件没10分钟会被检查一次，如果它们的总大小大于WAL_size_limit_MB，他会从最早的文件开始删除，直到总大小小于WAL_size_limit_MB。所有空白文件都会被删除
- 如果WAL_ttl_seconds不是0，而WAL_size_limit_MB是0，那么WAL日志会每WAL_ttl_seconds/2秒就被检查一次，事件晚于WAL_ttl_seconds秒的文件会被删除。
- 如果都不是零，WAL日志会每10分钟被检查一次，上面的两种检查都会被进行，而且ttl先进行。

## 其他信息

设置RocksDB：

- [设置配置]()
- 更多信息 [调优指南]()

关于rocksdb的实现可以在下面的文档找到

- [RocksDB概述与架构]()
- [不可修改Table文件的格式]()
- [log文件格式]()

