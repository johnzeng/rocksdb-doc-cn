RocksDB提供一组选项给用户来决定IO应该如何执行

# 控制写IO

## 范围Sync

RocksDB的数据文件通常通过追加的形式生成。文件系统会选择把写入缓冲起来直到脏页达到一个阈值，然后把所有这些页面都写出。这可能会造成突发写IO，导致线上的IO等待过久，导致高查询延迟。你可以要求RocksDB周期性通知OS把已经存在的脏页写出，具体做法就是为SST文件设置options.bytes_per_sync，为WAL文件设置options.wal_bytes_per_sync。在底层，每当一个文件到达这个大小的时候，他就会调用Linux的sync_file_range。当前使用的页不会被包含在范围Sync中。

## 限流器

你可以通过options.rate_limiter控制RocksDB的总写文件速率，一次保留足够的IO带宽给在线查询。参考[限流器]()。

## 最大写缓冲

往一个文件追加数据的时候，除非声明了需要fsync，否则RocksDB在写入文件系统前会有内部的文件缓冲区。这个缓冲区的最大大小可以通过options.writable_file_max_buffer_size来控制。在[直接IO模式]()或者在一个没有页缓存的文件系统，可以通过这个参数进行调优。在非直接IO模式，加大这个缓冲区的大小仅仅减少了write系统调用的次数，并且通常不会改变IO行为，所以，除非这个就是你想要的，通常应该把这个值设定为0来节省内存

# 控制读IO

## fadvise

当打开一个SST文件进行读取，用户设定options.advise_random_on_open = true(默认).可以决定RocksDB是否会使用FADV_RANDOM来调用fadvise。如果值为false，则打开文件的时候没有fadvise会被调用。如果主要的查询是Get或者是非常小范围的迭代，把这个选项设置为true通常更好，因为预读取在这些场景没什么帮助。另一方面，options.advise_random_on_open = false通过告知文件系统做底层预读取，通常会有更好的性能。

不幸的是，如果两种情况同时兼而有之，就没有一个好的设定了。有一个正在进行中的项目，希望通过给RocksDB内部迭代器设定预读取来解决这个问题。

## 压缩输入

压缩输入比较特别。他们是长的序列化读取，所以使用用户读取选项的fadvise选项通常不是特别好。另外，通常，尽管没有保证，压缩输入文件通常会很快被删除。RocksDB提供多个方式来解决这些问题：

### fadvise提示

RocksDB会对任何压缩输入文件根据options.access_hint_on_compaction_start调用fadvise。当一个文件被选为压缩输入的时候，这个可以覆盖随机fadvise设定。

### 给压缩输入使用不同的文件描述符

如果options.new_table_reader_for_compaction_inputs = true，RocksDB会使用不同的文件描述符来打开压缩输入。这可以避免混淆普通数据文件的fadvise设定和压缩输入文件的。这个选项的限制是，RocksDB不会仅仅创建一个新的文件描述符，而是重新读索引，过滤器和其他元数据块，然后把他们存储在内存，这会带来额外的IO以及更多的内存使用。

### 压缩输入文件的预读取

如果options.compaction_readahead_size为0，你可以自己做预读取。这个选项被设置的时候，options.new_table_reader_for_compaction_inputs会自动变为true。这个选项允许用户保持options.access_hint_on_compaction_start为NONE。

如果直接IO被打开，或者文件系统不支持预读取，设置这个是不好的。

# 直接IO

与上面的通过文件系统控制IO，你还可以通过option use_direct_reads与/或use_direct_io_for_flush_and_compaction，在RocksDB里面打开直接IO，来直接控制IO，如果直接IO被打开，上面的部分或者全部选项，都可能不生效。更多细节参考[直接IO]()

# 内存映射

options.allow_mmap_reads与options.allow_mmap_writes 允许rocksdb在读取或者写入的时候分别mmap整个数据文件，这种做法的好处是可以减少pried和write的时候的系统调用，并且，在很多情况下，可以减少内存拷贝。如果DB运行在ramfs，options.allow_mmap_reads通常可以显著提升性能。他们也可以在块设备的文件系统上被使用。然而，根据我们之前的经验，文件系统通常没法做到完美维护这类内存映射，有时候还会导致慢查询。在这种情况下，我们建议你只在必须的情况下，谨慎尝试。

