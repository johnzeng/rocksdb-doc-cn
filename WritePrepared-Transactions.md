RocksDB同时支持乐观和悲观两种并发控制。悲观事务使用锁来提供事务隔离。悲观事务的默认的写策略为WriteCommitted，也就是 只有在事务提交后，数据才写入db，或者说memtable。这个策略简化了实现，但是带来一些吞吐，事无大小，以及事务隔离级别多样性上的限制。下面，我们解释这些细节，然后展示其他写策略，WritePrepared和WriteUnprepared。然后我们会深入研究WritePrepared事务。

# WriteCommitted 优点与缺点

使用WriteCommitted策略，数据会在提交之后才写入到memtable。这极大地简化了读取逻辑，因为所有其他事务读到的数据都可以假设是已经提交的。然而，这个写策略，也意味着写入会被缓存到内存中。对于大事务，内存就会变成一个瓶颈。在2PC（两阶段提交）中，提交阶段的延迟同样变得引人注目，因为大多数的工作，如写memtable，会在提交阶段完成。当多个事务的提交以序列化方式完成，例如MySql的2PC实现，提交延迟会变成一个低吞吐主要的主要原因。更重要的是，这个写策略无法提供更弱的隔离等级，比如 *读未提交* ，对于某些应用（读未提交）可以提高吞吐。

# 备选方案：WritePrepared和WriteUnprepared

为了解决冗长的提交问题，我们应该在2PC的更早的阶段进行memtable的写入，这样提交阶段会变得更轻量，更快速。2PC由 写阶段，当transaction ::Put被调用的时候，  准备阶段，此时::Prepare被调用（DB承诺，后面如果要求提交事务，他就能提交了），以及 提交阶段， 这里::Commit 被调用，然后事务会被写入，并且对其他读者可见   组成（三个阶段，分别是写，准备，提交）。为了保证提交阶段轻量，memtable的写可以在::Prepare 或者 ::Put 阶段完成，他们分别产生了WritePrepared和WriteUnprepared两种写策略。带来的问题是，当另一个事务在读数据的时候，他需要一个办法来区分数据是不是已经提交，如果是，他们是不是在当前事务开始前提交的，比如说，在读事务的快照的时候。WritePrepared会有缓存数据的问题，对于大数据量的事务会构成内存瓶颈。但是他为从WriteCommitted迁移到WriteUnprepared提供了一个非常好的过渡。我们会在这了解释WritePrepared策略的设计。用于支持WriteUnprepared的设计可以参考[这里](WriteUnprepared-Transactions.md)

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



