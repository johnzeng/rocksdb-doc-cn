许多用户使用RocksDB实现了一个队列服务。在这些服务里，每个队列，新加入的内容会携带一个更大的序列号ID，而删除的时候，会删除最小的序列号ID。用户通常按序列号递增的顺序读取队列。

# Key编码

你可以简单的按照`<queue_id, sequence_id>`编码他，queue_id是按照固定长度编码的，而sequence_id则使用大端编码。

迭代key的时候，用户可以创建一个迭代器，找到`<queue_id, target_sequence_id>`，然后从该处开始迭代。

# 旧数据删除问题

由于最旧的数据要被删掉，在每个`queue_id`开始的地方，可能会有很大数量的”墓碑“。结果而言，下面两个查询会变得很昂贵：

- Seek(`<queue_id, 0>`)
- 当你在一个`queue_id`的最后一个seq ID的时候，尝试调用Next()

为了解决这个问题，你可以记住每个queue_id对应的第一个和最后一个seq ID，然后不要跨越这个范围进行迭代。

另一个办法是，当你在queue_id里面迭代的时候，可以在迭代中设置一个结束的key，通过让`ReadOptions.iterate_upper_bound`指向`<queue_id, max_int>``<queue_id+1>`。我们鼓励你总是设置这个，不管你是否因为删除而导致了慢查询。

# 检查一个queue_id的新seqID

如果一个用户处理完一个`queue_id`中的最后一个`sequenec_id`，然后拉取新插入的数据，只需要调用Seek(`<queue_id, last_processed_id>`)，然后调用Next()，然后看下一个key是否为同一个`queue_id`。确保`ReadOptions.iterate_upper_bound`指向`<queue_id+1>`以避免删除数据导致的慢查询。

如果你希望进一步优化这种使用场景，避免每次都二分搜索整个LSM树，考虑使用`TailingIterator`（或者在某些地方考虑`ForwardIterator`）[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h#L1235-L1241](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h#L1235-L1241)

# 更快回收删除数据的空间

队列服务是`CompactOnDeletionCollector`的一个很好的使用用例，他在压缩的时候会优先考虑更多的删除动作。给[这里](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/utilities/table_properties_collectors.h#L23-L27)定义的工厂类设置`immutableCFOptions::table_properties_collector_factories`

