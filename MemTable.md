MemTable是一个内存数据结构，他保存了落盘到SST文件前的数据。他同时服务于读和写——新的写入总是将数据插入到memtable，读取在查询SST文件前总是要查询memtable，因为memtable里面的数据总是更新的。一旦一个memtable被写满，他会变成不可修改的，并被一个新的memtable替换。一个后台线程会把这个memtable的内容落盘到一个SST文件，然后这个memtable就可以被销毁了。

影响memtable的最重要的几个选项是：

- memtable_factory: memtable对象的工厂。通过声明一个工厂对象，用户可以改变底层memtable的实现，并提供事先声明的选项。
- write_buffer_size：一个memtable的大小
- db_write_buffer_size：多个列族的memtable的大小总和。这可以用来管理memtable使用的总内存数。
- write_buffer_manager：除了声明memtable的总大小，用户还可以提供他们自己的写缓冲区管理器，用来控制总体的memtable使用量。这个选项会覆盖db_write_buffer_size
- max_write_buffer_number：内存中可以拥有刷盘到SST文件前的最大memtable数。

默认的memtable实现是基于skiplist的。除了默认的memtable实现，用户可以使用其他memtable实现，例如HashLinkList，HashSkipList或者Vector，以加快查询速度。

# Skiplist Memtable

基于Skiplist的memtable在多数情况下都有较好读，写，随机访问以及序列化扫描性能。除此之外，他还提供其他memtable没有的有用的功能，比如[并发插入]()以及[带Hint插入]()

# HashSkiplist Memtable

正如他们的名字暗示的，HashSkipList用一张哈希表组织数据，每个哈希桶内都是一个的skiplist，而HashLinkList则是用一张哈希表组织数据，每个哈希桶内则是使用一个排序好的链表。两种类型都是为了减少查询的时候的比较次数。一种好的使用例子是使用PlainTable SST格式结合他们，然后把数据存储在RAMFS里。

当做数据查询或者插入一个key的时候，目标key的前缀通过Options.prefix_extractor被提取出来，用于找到具体的哈希桶。在哈希桶里面，所有的比较都是完整（内部）key比较，跟SkipList的memtable一样。

使用基于哈希的memtable最大的限制就是做多个前缀扫描的时候需要拷贝和排序，这非常慢并且浪费内存

# 落盘

有三种场景会导致memtable落盘被触发：

- Memtable的大小在一次写入后超过write_buffer_size。
- 所有列族中的memtable大小超过db_write_buffer_size了，或者write_buffer_manager要求落盘。在这种场景，最大的memtable会被落盘
- WAL文件的总大小超过max_total_wal_size。在这个场景，有着最老数据的memtable会被落盘，这样才允许携带有跟这个memtable相关数据的WAL文件被删除。

就结果来说，memtable可能还没写满就落盘了。这是为什么生成的SST文件小于对应的memtable大小。压缩是另一个导致SST文件变小的原因，因为memtable里的数据是没有压缩的。

# 并发插入

如果不支持对memtable进行并发插入，从多个线程过来的并发写会按顺序应用到memtable中。并发memtable插入是默认打开的，可以通过allow_concurrent_memtable_write选项来关闭，尽管只有skip-list的memtable支持这个功能。

# 带Hint插入

# 原地更新
# 对比

|Memtable类型|SkipList|HashSkipList|HashLinkList|Vector|
|---|---|---|---|---|
|最佳使用场景|通用|带特殊key前缀的范围查询|带特殊key前缀，并且每个前缀都只有很小数量的行|大量随机写压力|
|索引类型|二分搜索|哈希+二分搜索|哈希+线性搜索|线性搜索|
|是否支持全量db有序扫描？|天然支持|非常耗费资源（拷贝以及排序一生成一个临时视图）|同HashSkipList|同HashSkipList|
|额外内存|平均（每个节点有多个指针）|高（哈希桶+非空桶的skiplist元数据+每个节点多个指针）|稍低（哈希桶+每个节点的指针）|低（vector尾部预分配的内存）|
|Memtable落盘|快速，以及固定数量的额外内存|慢，并且大量临时内存使用|同HashSkipList|同HashSkipList|
|并发插入|支持|不支持|不支持|不支持|
|带Hint插入|支持（在没有并发插入的时候）|不支持|不支持|不支持|


