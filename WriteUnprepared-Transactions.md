这份文档展示 为了在批量写请求还在写入的时候，把写memtable这个操作从准备阶段移动到未准备阶段 进行的最初设计。

```
WriteUnprepared很快就能成为可以在生产使用的功能了
```

# 目标

以下是这个项目的目标：

1. 减少内存消耗，这使得处理非常大的事务变为可能。
2. 避免由于一次性写入巨大的事务造成的写失速。

# 梗概

目前，事务会被缓存在内存，知道2PC的准备阶段。当事务非常巨大的时候，缓存的数据也会很大，并且一次性写入如此巨大的事务会对并行的事务造成负面影响。更进一步，缓存一个非常巨大的事务的所有数据会导致机器内存耗尽。对内存消耗最大的贡献者是 i)缓存的键值对，比如说，批量写  ii)每个key的锁。在这个设计，我们把工程拆分为三步，以减少内存。i)值 ii)键 iii)他们对应的锁

为了排除在巨大事务中用于缓存值使用的内存，未准备的数据会被逐步写入未预备批处理，并且在写批处理还在构建的时候就被写入数据库。然而，为了简化回滚算法，key还是被缓存起来了。当一个事务提交，每个未预备的批处理都会 更新 提交缓存中的一个项。这里主要的挑战是处理回滚，以及读取自己的写入。

事务需要可以读取他们自己的未提交事务。之前这是通过在搜索DB前，先查找缓存的数据来实现的。我们通过保持记录已经写入硬盘的未预备批量写的序列号，然后增加ReadCallback，让其在独到的数据的序列号能匹配的时候返回true，来解决这个问题。这只对巨大的事务生效，这些事务不需要让批量写请求缓存到内存。

目前，WritePrepared的回滚算法只能在没有存活快照的恢复流程之后工作。在WriteUnPrepared中，即使现在有快照，事务也可以回滚。我们这样设计回滚算法 i) 追加被事务修改key的值，在事务修改前的值，这样肯定可以取消写入 ii)提交回滚的事务。把一个回滚事务看做提交事务 极大简化了实现，因为现有的提交时处理存活快照的机制能与之无缝结合。WAL仍旧包括一个回滚标记，以保证在DB崩溃后，恢复过程能重新进行回滚。为了找到事务修改前的值，修改了的key的值必须被缓存在Transaction对象中。在没有缓存的批量写的情况下，在工程的第一阶段，我们仍需要缓存被修改的key的集合。在第二步，如果事务很大，我们从WAL中获取key集合。

RocksDB使用TransactionLockMgr追踪一个事务被锁定的key。对于巨大事务，被锁定的key的列表可能无法填入内存。自动锁定会上升至一个范围锁，用于近似一个集合的点锁。当RocksDB检测到一个事务正在锁定一个非常大的数量的在一个特定范围的key的时候，他会自动升级到区间锁。我们会在这个工程的第三阶段讨论这个。更多细节参考下面的“key锁”

# 阶段

计划是我们分三个阶段执行这个项目：

- 实现未预备批量写，但是key仍旧缓存在内存中，用于回滚和key上锁
- 回滚的时候，使用WAL来获取key集合，而不是从内存缓冲区获得。
- 对巨大的事务，实现区间锁机制。

# 实现

## WritePrepared概览

在已经有的WritePrepared策略，数据结构包括：

- PrepareHeap：一个正在处理中的准备序列号的堆
- CommitCache：一个从准备序列号到提交序列号的映射
	- max_evicted_seq_：从CommitCache淘汰的最大淘汰序列号
- OldCommitMap：从CommitCache淘汰的项，如果他们在某些线上快照还能看到的话，进入这里。
- DelayedPrepared：从PrepareHeap出来的小于max_evicted_seq_的准备序列号。

## Put

写入数据库需要使用批处理以避免因为写队列带来的额外消耗。为了避免与“批量写”冲突，他们会被成为“批量未预备(unprepared batches)”。通过批处理，我们还会节省未预备序列号的号码，这个号码我们需要生成，并且在我们的数据结构中追踪。一旦一个批处理到达一个可配置的阈值，他会被标记，以确定是不是有一个预备操作在WritePreparedTxn中，只不过一个新的WAL类型，名为BeginUnprepareXID，会被用于BeginPersistedPrepareXID（在WritePreparedTxn策略中被使用）的反面。所有的在同一个未预备批处理中的key会获得同样的序列号（除非有一个冲突的key，会把批处理拆分为多个子批处理）

提出的WAL格式： [BeginUnprepareXID] .... [EndPrepare(XID)]

这意味着，在最后准备之前，我们需要知道事务的XID（XID是一个事务生命周期中唯一的编码）。（这在Myrocks里面是这样的，因为我们在事务开始的时候生成XID）

Transaction对象会需要追踪已经写入到db的未预备事务的列表。为此，Transaction对象会包含一系列unprep_seq数字，当一个未预备批处理被写入，unprep_seq会被加入到这个集合。

在未预备批量写中，unprep_seq数字同样会被加入到未预备堆（类似WritePreparedTxn的预备堆）

## prepare

在prepare的时候，我们会把当前批量写中剩余的项写出到WAL，但是使用BeginPersistedPrepareXID来表示这个事务已经准备好。这样，在崩溃的时候，我们可以给应用返回已经准备好的事务，这样应用可以进行正确的操作。未预备事务会在恢复的时候隐式地回滚。

提出的WAL格式： [BeginUnprepareXID]...[EndPrepare(XID)] ... [BeginUnprepareXID]...[EndPrepare(XID)] ... [BeginPersistedPrepareXID]...[EndPrepare(XID)] ... ...[Commit(XID)]

在这种情况下，最后的准备获得BeginPersistedPrepareXID，而不是BeginUnprepareXID，以表示事务真的准备好了。

注意，尽管DB（sst文件）在WritePreparedTxn和WriteUnpreparedTxn是向前向后兼容的，WriteUnpreparedTxn的WAL对WritePreparedTxn不是想前兼容的：WritePreparedTxn肯定会在回复一个通过WriteUnpreparedTxn生成的WAL的时候失败，因为有新的标记类型。然而，WriteUnpreparedTxn仍旧向后兼容WritePreparedTxn，并且可以读取WritePreparedTxn的WAL，因为他们看起来是一样的。

## Commit

提交的时候，提交映射和未预备堆需要被更新。对于WriteUnprepared，一个提交会潜在地拥有多个预备序列号。所有(unprep_seq, commit_seq)对都需要被加入到提交映射，并且所有unprep_seq都必须从unprepare_heap删除。

如果提交在没有准备的时候被执行，并且事务没有提前写入未预备批处理，那么当前的未预备批量写会直接写到类似WritePreparedTxn的CommitWithoutPrepareInternal的情况中。如果事务已经写入为预备批处理，那么我们认为预备阶段也被加入。

## Rollback

在WritePreparedTxn中，回滚实现被限制在只在恢复之后进行回滚。他大概是这样实现的：

1.



