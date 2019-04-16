# 文件结构

磁盘上的文件被分成多层进行组织。我们叫他们Level-1, Level-2，等等，或者简单的L1,L2，等等。一个特殊的层，Level-0(L0)，会包含刚从内存memtable落盘的数据。每个层（除了Level0）都是一个独立的排序结果

![level_structure](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/level_structure.png)

在每一层（除了Level-0），数据被切分成多个SST文件。

![level_files](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/level_files.png)

每一层都是一个排序结果，因为每个SST文件中的key都是排好序的（参考[基于块的表格式]())。如果需要定位一个key，我们先二分查找所有文件的起始和结束key，定位哪个文件有这个key，然后二分查找具体的文件，来定为key的位置。总的来说，就是在该层的所有key里面进行二分查找。

所有非0层都有目标大小。压缩的目标是限制这些层的数据大小。大小目标通常是指数增加。

![level_targets](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/level_targets.png)

# 压缩(compaction)

当L0的文件数量到达level0_file_num_compaction_trigger，压缩就(compaction)会被触发，L0的文件会被合并进L1。通常我们需要把所有L0的文件都选上，因为他们通常会有交集：

![pre_l0_compaction](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l0_compaction.png)

压缩过后，可能会使得L1的大小超过目标大小：

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l0_compaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l0_compaction.png)

这个时候，我们会选择至少一个文件，然后把它跟L2有交集的部分进行合并。生成的文件会放在L2：

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l1_compaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l1_compaction.png)

如果结果仍旧超出下一层的目标大小，我们重复之前的操作 —— 选一个文件然后把它合并到下一层:

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l1_compaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l1_compaction.png)

然后

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l2_compaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l2_compaction.png)

然后

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l2_compaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l2_compaction.png)

如果有必要，多个压缩会并发进行：

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/multi_thread_compaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/multi_thread_compaction.png)

最大同时进行的压缩数由max_background_compactions控制。

然而，L0到L1的压缩不能并行。在某些情况，他可能变成压缩速度的瓶颈。在这种情况下，用户可以设置max_subcompactions为大于1。在这种情况下，我们尝试进行分片然后使用多线程来执行。

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/subcompaction.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/subcompaction.png)

# 压缩文件选择

当多个层触发压缩条件，RocksDB需要选择哪个层先进行压缩。对每个层通过下面方式生成一个分数：

- 对于非0层，分数是当前层的总大小除以目标大小。如果已经有文件被选择进行压缩到下一层，这些文件的大小不会算入总大小，因为他们马上就要消失了
- 对于level0，分数是文件的总数量，除以level0_file_num_compaction_trigger，或者总大小除以max_bytes_for_level_base，这个数字可能更大一些。（如果文件的大小小于level0_file_num_compaction_trigger，level 0 不会触发压缩，不管这个分数有多大）

我们比较每一层的分数，分数高的压缩优先级更高。

具体那个一个文件会被压缩会在[选择层压缩文件]()中解释。

# 层的目标大小

## level_compaction_dynamic_level_bytes为false

如果level_compaction_dynamic_level_bytes为false，那么层目标这样决定： L1的目标是max_bytes_for_level_base，然后Target_Size(Ln+1) = Target_Size(Ln) * max_bytes_for_level_multiplier * max_bytes_for_level_multiplier_additional[n]。max_bytes_for_level_multiplier_additional默认全为1。

举个例子，如果max_bytes_for_level_base = 16384，max_bytes_for_level_multiplier = 10 ，max_bytes_for_level_multiplier_additional没有被设置，那么L1,L2,L3与L4层的大小分别是16384, 163840, 1638400, 以及 16384000。

## level_compaction_dynamic_level_bytes为true

最后一层的目标大小(层数-1)总是层的实际大小。剩下的Target_Size(Ln-1) = Target_Size(Ln) / max_bytes_for_level_multiplier。如果那一层的大小会小于max_bytes_for_level_base / max_bytes_for_level_multiplier，我们不会填充他。这些层会保持为空，所有L0的压缩会跳过这些层，直接到第一个符合大小的层。

举个例子，如果max_bytes_for_level_base为1GB，num_levels=6，最后一层的实际大小为276GB，那么L1-L6层的目标大小经分别为0，0，0.276GB,2.76GB,27.6GB以及276GB。

这是为了保证一个稳定的LSM树结构，如果level_compaction_dynamic_level_bytes为false，就无法保证。举个例子，在前面的例子中：

![https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/dynamic_level.png](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/dynamic_level.png)

我们可以保证90%的数据存储在最后一层，9%的数据在倒数第二层。这样会有许多好处。

# TTL

如果没有人更新一个文件中key，那么这个文件讲不会进入压缩流程，而这个文件就会存活很久。例如，在某些特定场景，key会被“软删除” —— 把数值设置为空而不是直接使用Delete删除。这个“已经删除”的key范围可能不会有任何新的写入了，这样，这个数据就会在LSM里面待很长时间，浪费空间。

一个动态ttl列族选项被用于解决这个问题。文件（或者说，数据）老于TTL的数据在没有其他后台任务的时候会被定时压缩。这回让数据进入常规的压缩流程，摆脱无用的老数据。对于所有不是最底层的，比ttl要新的数据，以及所有在最底层老于ttl的数据，这还有一个（好的）副作用。注意这会导致更多的写因为RocksDB会调度更多的压缩。

