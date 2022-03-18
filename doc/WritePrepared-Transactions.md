RocksDB同时支持乐观和悲观两种并发控制。悲观事务使用锁来提供事务隔离。悲观事务的默认的写策略为WriteCommitted，也就是 只有在事务提交后，数据才写入db，或者说memtable。这个策略简化了实现，但是带来一些吞吐，事无大小，以及事务隔离级别多样性上的限制。下面，我们解释这些细节，然后展示其他写策略，WritePrepared和WriteUnprepared。然后我们会深入研究WritePrepared事务。

# WriteCommitted 优点与缺点

使用WriteCommitted策略，数据会在提交之后才写入到memtable。这极大地简化了读取逻辑，因为所有其他事务读到的数据都可以假设是已经提交的。然而，这个写策略，也意味着写入会被缓存到内存中。对于大事务，内存就会变成一个瓶颈。在2PC（两阶段提交）中，提交阶段的延迟同样变得引人注目，因为大多数的工作，如写memtable，会在提交阶段完成。当多个事务的提交以序列化方式完成，例如MySql的2PC实现，提交延迟会变成一个低吞吐主要的主要原因。更重要的是，这个写策略无法提供更弱的隔离等级，比如 *读未提交* ，对于某些应用（读未提交）可以提高吞吐。

# 备选方案：WritePrepared和WriteUnprepared

为了解决冗长的提交问题，我们应该在2PC的更早的阶段进行memtable的写入，这样提交阶段会变得更轻量，更快速。2PC由写阶段，当transaction ::Put被调用的时候，准备阶段，此时::Prepare被调用（DB承诺，后面如果要求提交事务，他就能提交了），以及提交阶段， 这里::Commit 被调用，然后事务会被写入，并且对其他读者可见组成（三个阶段，分别是写，准备，提交）。为了保证提交阶段轻量，memtable的写可以在::Prepare 或者 ::Put 阶段完成，他们分别产生了WritePrepared和WriteUnprepared两种写策略。带来的问题是，当另一个事务在读数据的时候，他需要一个办法来区分数据是不是已经提交，如果是，他们是不是在当前事务开始前提交的，比如说，在读事务的快照的时候。WritePrepared会有缓存数据的问题，对于大数据量的事务会构成内存瓶颈。但是他为从WriteCommitted迁移到WriteUnprepared提供了一个非常好的过渡。我们会在这了解释WritePrepared策略的设计。用于支持WriteUnprepared的设计可以参考[这里](WriteUnprepared-Transactions.md)

# 简单聊聊WritePrepared

这些是需要解决的主要的设计问题：

- 我们如何确定一个DB里的键值对是由那个事务写的？
- 我们如何确定事务Txn_w写的一个键值对 出现在一个正在读的事务Txn_r的读快照中？
- 我们如何回滚废弃事务的数据？

使用WritePrepared的时候，一个事物仍旧需要缓冲写入的批量写对象。当2PC的::Prepare被调用，他把内存的批量写请求写入WAL，同时写入到memtable（每个列族一个memtable）；我们复用已有的RocksDB的序列号（seq_no，prepare_seq）来标记同一个批量写请求的所有键值对，同事也被用于事务的标识号码。提交的时候，他写一个提交标记到WAL，他的序列号，commit_seq，会被用作事物的提交时间戳。在把提交序列号释放给读取器前，他在内存的数据结构中存储一个从prepare_seq到commit_seq的映射，命名为CommitCache。当一个事务从DB读取数据，（使用prepare_seq标记），它使用CommitCache来确定commit_seq对应的值是否在他的读快照内。为了回滚一个丢弃的事务，我们通过写入丢弃事务之前的状态，来恢复事务前的状态。

CommitCache是一个无所数据结构，用于缓存最近的提交记录。对于大多数提交及时的事务，检索这个缓存的数据是足够的。当我们需要在缓存中丢弃旧记录的时候，他仍旧维护其他数据结构来保证一些花费很多时间的事务的特殊情况。我们会在下面的一些细节覆盖到这些场景。

# WritePrepared设计

这里我们展现WritePrepared事务的设计细节。我们从展示CommitCache的搞笑设计开始，然后深入到其他数据结构，因为我们认为他们是在CommitCache之上保证正确性的重要存在。

## CommitCache

一份数据是否提交的问题，主要受最近的事务的影响。换句话说，给定一个合适的原地回滚算法，我们可以假设任何在DB里的老数据都是提交的，并且已经出现在了一个正在读取的，最近生成的，事务快照中。这中间的优劣很明显，维护一个最近提交记录的缓冲已经对大多数情况都是够用了。CommitCache就是为了这个目的设计的一个高效数据结构。

CommitCache是一个定长，内存数组，记录提交数据。为了使用一个提交记录更新这个缓存，我们先使用数组大小索引`prepare_seq`，然后把数组中对应的项目重写，比如CommitCache[ `prepare_seq` % `array_size`] = <`prepare_seq`, `commit_seq`> 。每一个插入会导致一条记录被淘汰，导致`max_evicted_seq`,CommitCache中的最大淘汰序列号，被更新。当查找CommitCache中的数据的时候，如果`prepare_seq` > `max_evicted_seq`且`prepare_seq`不再缓存中，那么他就没有被提交。如果有一个项目没有在缓存中找到，那么他就是提交了，并且可以被满足快照序列号`snap_seq`  >= `commit_seq` 的 事务读取。如果`prepare_seq` < `max_evicted_seq`，那么我们正在读取一个老数据，除非有证据显示没有提交，否则大多数都是已经提交的，我们会在下方进行详细解释。

对于一个 80K tps（每秒事务数）的写操作，提交缓存中有8M个项目（在TransactionDBOptions::wp_commit_cache_bits中硬编码），并且每个事务的事务号每次加2，会花费大概50秒来使一个CommitCache中的项被淘汰。然而， 在实际使用中，准备和提交之间的延迟一般不到1毫秒，因此大多数情况都足够用了。然而，为了保证正确性，我们需要覆盖到准备好的事务在`max_evicted_seq`超过`prepare_seq`的情况，否则正在进行读取的事务会认为他已经被提交了。为了保证这个，我们维护一个准备序号的堆，名为PreparedHeap：一个`prepare_seq`会在`::Prepare`的时候被插入，然后在`::Commit`的时候被删除。当`max_evicted_seq`增加，如果他比堆里面最小的`prepare_seq`还要大，我们会弹出这个项，然后把它存储到一个名为`delayed_prepared_`的set中。检查`delayed_prepared_`是否为空是一个非常高效的操作，我们需要在把一个旧的`prepare_seq`称为已提交前进行这个操作。否则，正在读取的事务应该检查`delayed_prepared_`来确定他们正在读取的`prepare_seq`是不是会在这里面被发现。我们需要强调，这种场景在常规情况下都不应该发生，因此不会对性能有负面影响。

尽管对于读-写事务，他们应该在`::Prepare`阶段之后不到1毫秒的时间内提交，还是有可能有一些只读事务，会持有一些非常老的快照。比如说，备份数据的时候，使用了一个事务，可能需要好几个小时才进行完成。这种只读事务不能假设一个旧数据进入他们正在读的快照中，因此他们的快照可能也会很老。更准确的说，如果有一个存活快照，拥有`snap_seq`满足`prepare_seq`<= `snap_seq` < `commit_seq`，我们仍旧需要保存一个淘汰了的项目，他的seq<`prepare_seq`，`commit_seq` >所有。在这种情况，我们把`prepare_seq`插入到`old_commit_map_`，一个从快照序列号映射到一个`prepare_seq`集合的map。唯一需要浪费时间查询这个数据结构的事务，是那些正在读取一个非常老的快照的事务。这个数据结构的大小应该是非常小的，因为 i) 只有很少的事务在进行备份 ii) 只有有限数量的并行事务会在读取的快照的时候有交错的事务。`old_commit_map_`会在快照通过周期性为`max_evicted_seq_`增长而进行处理，触发快照列表拉取的时候，因为快照被DB告知需要释放，而被惰性垃圾回收。


## PreparedHeap

就如他的名字所说，PreparedHeap是一个prepare_seq的堆：一个prepare_seq会在 ::Prepare的时候被插入，然后在::Commit的时候被移除。根据上面的内容，这个堆结构允许高效检索针对max_evicted_seq的最小的prepare_seq。为了从堆里面保证高效删除数据，实际的删除动作会延迟到数据到达堆顶的时候，也就是，他是最小的一项的时候。为了做到这个，如果某个待删除的数据不在对顶，这个项会被写到另一个堆。每次修改，两个堆的顶都需要进行比较，以检查当前主堆的顶是不是被标记为已经删除。

## 回滚

为了回滚一个事务，对于每个已经写入的key/value对，我们写入另外的key/value对来取消之前的写入。感谢通过锁实现的悲观事务实现的 写-写 冲突避免，我们可以保证每个key挂起的写请求只有一个，意味着我们只需要查看每个key的前一个值就可以找到我们需要恢复到的值了。如果::Get的结果来自一个最大序列号为kMaxSequenceNumber的快照，返回的序列号是一个正常数字（即将回滚的数据会被跳过，因为他们没有被提交），然后我们对该key执行::Put，如果这个key不存在，我们插入一个::Delete数据。所有的数值（不管是废弃的事务还是回滚的值）之后会被以一个相同的commit_seq被提交，然后废弃事务的prepare_seq会被从PreparedHeap中移除。

## 原子提交

在一个提交过程中，一个提交标记会被写入到WAL，然后一个提交项会被增加到CommitCache。这两个事情需要被原子地完成，否则某个时间点的一个读事务可能会错过CommitCache的更新，但是过一阵子又看到了。我们通过先更新CommitCache，然后再发布提交项的序列号来解决这个问题。这样，如果一个读取中的快照可以看到提交序列号，那么可以保证CommitCache也已经更新。这是通过 ::WriteImpl 中的一个PreReleaseCallback调用来实现的。PreReleaseCallback也被用于把prepare_seq加到PreparedHeap，这样他的顶总是最小的未提交事务（参考 *最小未提交* 章节看这个是如何使用的）。

当我们有两个写请求队列（two_write_queues=true），那么主写队列可以同时向WAL和memtable写，而第二写队列只能写WAL，这个会被用于在WritePrepared事务中写提交标记。在这个场景下，主队列（以及他的PreReleaseCallback回调）总是用于准备项，而第二队列（以及他的PreReleaseCallback回调）总是用于提交。这样 i)避免两个队列的竞争 ii)维护了PreparedHeap的顺序增加，以及 iii)简化代码，避免并发插入到CommitCache（以及从中淘汰数据的代码）。

由于两个队列都能在另一个没有使用完预分配的较小的序列号的时候增加他们的最后序列号，这时候可能会构成一个原子化问题。为了解决这个问题，我们引入了最后发布的序列号这个概念，会在创建快照的时候使用。当我们有一个写队列的时候，他跟最后的序列号一样。当我们有两个队列的时候，这就是最后提交了的项目（有第二个队列执行）。这个措施会影响到非2PC事务，因为他们会被切分为两步： i)通过主队列写memtable， ii)通过第二队列提交并发布序列号。

## IsInSnapshot

IsInSnapshot(prepare_seq, snapshot_seq) 实现了WritePrepared的核心算法，把所有数据结构放在一起，然后决定一个标记为prepare_seq的值在snapshot_seq的快照中是否能被读取。

这里是IsInSnapshot算法的简化版：

```
inline bool IsInSnapshot(uint64_t prep_seq, uint64_t snapshot_seq,
                         uint64_t min_uncommitted = 0,
                         bool *snap_released = nullptr) const {
  if (snapshot_seq < prep_seq) return false;
  if (prep_seq < min_uncommitted) return true;
  max_evicted_seq_ub = max_evicted_seq_.load();
  some_are_delayed = delayed_prepared_ not empty
  if (prep_seq in CommitCache) return CommitCache[prep_seq] <= snapshot_seq;
  if (max_evicted_seq_ub < prep_seq) return false; // still prepared
  if (some_are_delayed) {
     ...
  }
  if (max_evicted_seq_ub < snapshot_seq) return true; // old commit with no overlap with snapshot_seq
  // commit is old so is the snapshot, check if there was an overlap
  if (snaoshot_seq not in old_commit_map_) {
    *snap_released = true;
    return true;
  }
  bool overlapped = prepare_seq in old_commit_map_[snaoshot_seq];
  return !overlapped;
}
```

如果他能确认commit_seq <= snapshot_seq，那么他返回true，否则，他返回false。

- snapshot_seq < prep_seq 推导出 commit_seq > snapshot_seq ，因为 prep_seq <= commit_seq
-	prep_seq < min_uncommitted 推导出 commit_seq <= snapshot_seq
-	检查CommitCache之前 先检查some_are_delayed中的delayed_prepared_是否为空，这个优化用于在没有延迟的事务的时候（通常如此），跳过锁申请。
- 如果不在CommitCache里，且延迟的准备申请, 那么这就是一个老的提交，已经从CommitCache被淘汰了
	- 	max_evicted_seq_ < snapshot_seq 那么 commit_seq < snapshot_seq 因为 commit_seq <= max_evicted_seq_
	-	否则, old_commit_map_ 包含所有那些老快照和所有覆盖他们的提交

下面我们看一下IsInSnapshot的完整的实现，覆盖一些边角场景。

```cpp
inline bool IsInSnapshot(uint64_t prep_seq, uint64_t snapshot_seq,
                         uint64_t min_uncommitted = 0,
                         bool *snap_released = nullptr) const {
  if (snapshot_seq < prep_seq) return false;
  if (prep_seq < min_uncommitted) return true;
  do {
    max_evicted_seq_lb = max_evicted_seq_.load();
    some_are_delayed = delayed_prepared_ not empty
    if (prep_seq in CommitCache) return CommitCache[prep_seq] <= snapshot_seq;
    max_evicted_seq_ub = max_evicted_seq_.load();
    if (max_evicted_seq_lb != max_evicted_seq_ub) continue;
    if (max_evicted_seq_ub < prep_seq) return false; // still prepared
    if (some_are_delayed) {
      if (prep_seq in delayed_prepared_) {
        // might be committed but not added to commit cache yet
        if (prep_seq not in delayed_prepared_commits_) return false;
        return delayed_prepared_commits_[prep_seq] < snapshot_seq;
      } else {
        // 2nd probe due to non-atomic commit cache and delayed_prepared_
        if (prep_seq in CommitCache) return CommitCache[prep_seq] <= snapshot_seq;
        max_evicted_seq_ub = max_evicted_seq_.load();
      }
    }
  } while (UNLIKELY(max_evicted_seq_lb != max_evicted_seq_ub));
  if (max_evicted_seq_ub < snapshot_seq) return true; // old commit with no overlap with snapshot_seq
  // commit is old so is the snapshot, check if there was an overlap
  if (snaoshot_seq not in old_commit_map_) {
    *snap_released = true;
    return true;
  }
  bool overlapped = prepare_seq in old_commit_map_[snaoshot_seq];
  return !overlapped;
}
```

- 由于max_evicted_seq_和CommitCache分别更新，使用while循环，通过确保max_evicted_seq_在CommitCache搜索过程中没被修改，来达到简化算法的目的。
- 一个延迟的已经准备好的事务的提交涉及到四步：i)更新CommitCache ii)加入到delayed_prepared_commits_ iii)发布序列号 iv)从delayed_prepared_中删除。
	- 如果读者只 遵从 CommitCache搜索+delayed_prepared_搜索这个要求，他可能发现一个延迟了的准备好的事务在上面两个都不存在，然后没有检查他的commit_seq。为了解决这个问题，如果序列号没有在delayed_prepared_中，他需要再一次检索CommitCache。预留的序列号保证，如果真的提交了，他会看到提交。
	- 有一些奇怪的场景，一个延迟提交的事务可以在他的数据从delayed_prepared_列表中删除前，被从提交缓存中淘汰。delayed_prepared_commits_，会在每次一个延迟准备好的数据在从提交缓存中淘汰的时候，帮助我们不要丢调这些提交。

##	落盘/压缩

落盘/压缩线程，类似于读事务，使用IsInSnapshot API来确定那个version可以被安全地垃圾回收，而不会影响到线上快照。区别在于，在压缩调用IsInSnapshot的时候，一个快照可能已经释放。为了解决这个问题，如果IsInSnapshot带有拓展的snap_released参数，那么，如果他不能很确定的返回true、false结果，他会给调用者发信号，告知snapshot_seq不再有效。

## 重复的key

WritePrepared会把一个批量写请求中的所有数据都是用同一个序列号写入。这就假设了一个批量写中没有重复的key。为了处理重复的key，我们把一个批量写请求拆分成多个子批处理，每出现一个重复的key切分一次。如果有一个key有相同的序列号，memtable返回false。然后memtable的插入者增加序列号，然后再试一次。

这个实现的限制是，写线程需要预先知道子批处理的数量，这样他可以根据需要，给每个写入预分配序列号。当使用事务API，这就可以通过 WriteBatchWithIndex的索引 以很小的代价来解决，因为这个东西他有一个机制来删除重复的key。然而，当调用::CommitBatch来写入一批数据到DB的时候，DB需要迭代整个批量写请求，然后计算拆分出来的子批处理数量。如果有很多这样的写请求，他的影响就比较可观了。

探测重复的key需要知道列族的比较器。如果一个列族被丢弃了，我们需要先确保WAL不含有一个属于该列族的项。否则，如果DB之后崩溃了，恢复过程会看到这个记录，但是无法通过一个比较器来查看这个是不是重复的数据。


# 优化

这里我们覆盖一些额外的细节，主要关于我们在提升性能方面做得设计。

## 无锁CommitCache

大多数从最近数据读取的请求都会进入CommitCache。因此，保证CommitCache读的性能非常重要。为了这个目的，使用一个定长的数组是一个明智的决定。然而我们需要做更多的事情，来避免读请求的同步带来额外的消耗。为此，我们让CommitCache变成一个std::atomic<uint64_t>的数组，并且把<prepare_seq, commit_seq>编码为64bit数字。从数组的读和写分别通过std::memory_order_acquire 和 std::memory_order_release来完成。在一个x86_64架构的机器里，由于硬件的缓存一致性保证，这些操作会被编译为简单的内存读写。对于这个数据结构，我们未来会探索其他的设计。

为了把<prepare_seq, commit_seq>编码为64bit，我们使用这个算法

- prepare_seq的高位已经通过CommitCache的索引来声明了
- prepare_seq的低位被编码到64bit数字的高位。
- commit_seq和prepare_seq的差异部分会被编码到64bit数字的低位

## 减少max_evicted_seq的更新

通常max_evicted_seq被期望为在每次CommitCache有数据淘汰就更新。尽管更新max_evicted_seq并不会非常大的开销，但是维护他的操作会带来开销。例如，他需要持有一个互斥锁，来检查PreparedHeap的顶（尽管这个可以优化为无互斥锁）。更重要的是，他涉及到获取db的互斥锁来获取db的存活快照，因为维护old_commit_map_依赖于 到max_evicted_seq为止的存活快照列表。为了减小开销，每次更新max_evicted_seq，我们增加1% CommitCache大小的值（如果他没有超过最后一次发布的序列号数字），这样，在CommitCache数组回卷前，我们只需要维护100次淘汰就可以了。

## 无锁快照列表

上面，我们提到了一些只读快照，用于进行备份，他们应该指定到特定时间点。在插入数据的时候，每次淘汰数据都会需要检查存活的快照（在max_evicted_seq最后一次增加的时候产生的）。因此他发生的非常频繁，他需要更高的执行效率，不能带互斥锁。我们因此设计了一个数据结构，允许我们使用一个无锁的检查。为此，我们存储最开始的S个快照（S被硬编码在了TransactionDBOptions::wp_snaoshot_cache_bits，为128）到一个std::atomic<uint64_t>的数组中，我们知道这个在x86_64架构中有非常高的读写效率。一个独立的写进程 使用一个 从序号0开始按递增顺序排列好的 的快照列表 更新这个数组，然后原子化更新他的大小。

更新后，我们需要保证并行的读者可以读到所有有效的快照。新的和旧的数组都会排序号，新的列表是之前的列表的一个子集增加一些新的数据。因此如果一个快照在新和旧的列表都出现了，他会跟在新的数组中以一个更低的索引出现。所以如果我们简单地按顺序插入快照，如果有覆盖的项，在新的列表还是有效的，要么他就被写到了数组的同一个地方，要么他在被其他东西覆盖前，写到一个索引更小的地方。这保证了一个从这个数组读数据的读者最终会看到一个快照，在被覆盖前，或者覆盖后，重复出现在更新中。

如果快照的数量超过数组大小，剩余的更新会被存储到一个被互斥锁保护的vector中。这只是为了保证一些边角案例的正确性，我们在正常运行中不应该出现这种场景。

## 最小未提交

我们持续追踪最小的未提交数据，并把它存储到快照中。当读取数据的时候，如果他的序列号小于最小未提交数据，我们跳过在CommitCache中搜索数据，以减小cpu缓存未命中率。如果delayed_prepared_非空，则其中最小的项会代表*最小未提交*。否则，通常，PreparedHeap中的顶部元素会代表最小未提交，我们需要感谢 值在主写队列中，按照递增序列增加数据的规则。 *最小未提交*同时会被用于限制ValidateSnapshot对memtable的搜索，当他知道*最小未提交*已经大于memtable的最小序列号的时候，就不用搜索了。

## rocksdb_commit_time_batch_for_recovery

正如上面解释的，我们已经声明了提交操作只会在第二队列发生。然而，当该提交操作伴随CommitTimeWriteBatch出现的时候，他会同时写入memtable，他最终会发送这个批写入到主队列。在这种场景，我们需要做两次独立的写操作来完成提交：第一步，通过主队列写入CommitTimeWriteBatch，然后第二步，通过第二队列提交数据和准备好的批操作。为了避免这个额外操作，用户可以设置rocksdb_commit_time_batch_for_recovery配置变量为true，告知rocksdb这些数据只在恢复的时候需要用到。Rocksdb只需要写CommitTimeWriteBatch到WAL就行了。他仍旧保留最后的拷贝在内存中，以便在flush之后写入到SST文件。当时用这个选项时，CommitTimeWriteBatch不能有重复的项，因为我们不希望对每次提交请求浪费计算子批处理的时间。

# 实验结果

这里有一些性能优化测试结果的汇总（通过MyRocks得到）。

- benchmark...........tps.........p95 latency....cpu/query
- insert...................68%
- update-noindex...30%......38%
- update-index.......61%.......28%
- read-write............6%........3.5%
- read-only...........-1.2%.....-1.8%
- linkbench.............1.9%......+overall........0.6%

这里有结果的详细内容 [内存Sysbench](https://gist.github.com/maysamyabandeh/bdb868091b2929a6d938615fdcf58424) 和[SSD Sysbench](https://gist.github.com/maysamyabandeh/ff94f378ab48925025c34c47eff99306)

# 现有限制

读请求有大概1%左右的性能消耗。这是因为需要额外花费功夫来确定一个数据是否已经提交。

合并操作的回滚操作目前无法打开。因为在MyRocks中，对key使用合并前没有加锁。

目前不支持Iterator::Refresh。没有基础性的障碍，以后可以根据需要添加进来。

尽管使用WritePrepared策略生成的DB可以向前、向后兼容传统的WriteCommitted策略，但是WAL格式不能兼容。因此，为了改变WritePolicy，WAL需要先通过落盘进行清空。

如果two_write_queues被打开，非2PC事务会需要两个写请求来完成。如果主要的事务都是非2PC，two_write_queues选项应该被关闭。

TransactionDB::Write需要额外的开销来探测批量写请求中的重复的key。

如果一个列族被删除，WAL需要提前被清理，这样WAL中就没有该列族的数据了。

当使用rocksdb_commit_time_batch_for_recovery时，传递给CommitTimeWriteBatch的数据不能有重复的key，并且只有在一个memtable落盘之后才能被看到。


