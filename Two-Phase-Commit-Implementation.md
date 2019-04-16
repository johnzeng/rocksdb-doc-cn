这个文档简要解析了Rocksdb的两阶段提交的实现。

这个工程会被分解为五个关注部分：

- 修改WAL格式
- 拓展已有的事务API
- 修改写流程
- 修改恢复流程
- 与MyRocks整合

# 修改WAL日志

WAL包含一个或多条日志。每个日志都是一个或者更多序列化的WriteBatches。恢复过程中，WriteBatches会通过日志重新构建。为了修改WAL格式或者拓展他的功能，我们只需要考虑我们的WriteBatches。

一个WriteBatches是一个排序好的记录集合（Put(k,v), Merge(k,v), Delete(k), SingleDelete(k)），他们代表了RocksDB的写操作。每个记录有一个二进制字符串来表示。记录加入到一个WriteBatch的时候，他们的二进制表示内容会被追加到WriteBatch的二进制字符串表示后面。这个二进制字符串的前缀是一个批处理的开始序列号，之后是批处理中记录的数量。如果操作不是应用于default列族，那么每个记录可能会有一个列族修改记录作为前缀。

一个WriteBatch可以通过拓展WriteBatch::Handler被遍历。MemTableInserter是一个WriteBatch::Handler 的拓展，他把一个WriteBatch包含的操作插入到正确的列族的Memtable。

一个已有的WriteBatch可能有以下逻辑表示：

Sequence(0);NumRecords(3);Put(a,1);Merge(a,1);Delete(a);

为了实现2PC，对WriteBatch格式的修改包括增加四个新的记录。

- Prepare(xid)
- EndPrepare(xid)
- Commit(xid)
- Rollback(xid)

一个支持2PC的WriteBatch可能有以下逻辑表示：

Sequence(0);NumRecords(6);Prepare(foo);Put(a,b);Put(x,y);EndPrepare();Put(j,k);Commit(foo);

可以看到Prepare(xid) 和 EndPrepare()的关系有点类似于括号，会包含ID为'foo'的事务的操作。Commit(xid)和Rollback(xid)标记表示ID为xid的事务的操作需要被提交或者回滚。

序列ID分布

当一个WriteBatch被插入到一个memtable(通过MemTableInserter插入)，每个操作的序列ID等于WriteBatch的序列ID 加上 这个WriteBatch之前的记录消耗的的序列号。这个隐式的WriteBatch 序列号ID映射在2PC加入后将不再存在。在于给Prepare()中包含的操作会消耗序列号ID，方式是以相对Commit()标记的相对的位置进行消耗。这个Commit()标记可能在另一个WriteBatch或者来自他执行准备操作的日志。

向后兼容

WAL格式没有版本号，所以我们需要注意向后兼容。一个当前版本的RocksDB不能回复一个带有2PC标记的WAL日志。实际上他可能会因为无法识别记录id而崩溃。然而，这不重要，只需要给当前版本的RocksDB打补丁让他可以在遇到新的WAL格式的时候跳过prepared节和未知标记即可。

当前进度

参考 [这里](https://reviews.facebook.net/D54093)

# 拓展Transaction API

我们现阶段只关注悲观事务的2PC。客户端必须提前声明他们是否需要使用2PC语义。例如，客户端代码可能是这样的：

```
TransactionDB* db;
TransactionDB::Open(Options(), TransactionDBOptions(), "foodb", &db);

TransactionOptions txn_options;
txn_options.two_phase_commit = tr
txn_options.xid = "12345";
Transaction* txn = db->BeginTransaction(write_options, txn_options);
    
txn->Put(...);
txn->Prepare();
txn->Commit();
```

一个事务对象现在拥有有更多的状态，所以我们修改状态的枚举：

```
enum ExecutionStatus {
  STARTED = 0,
  AWAITING_PREPARE = 1,
  PREPARED = 2,
  AWAITING_COMMIT = 3,
  COMMITED = 4,
  AWAITING_ROLLBACK = 5,
  ROLLEDBACK = 6,
  LOCKS_STOLEN = 7,
};
```

事务API会有一个新的成员函数Prepare()。Prepare()会调用WriteImpl，把他自身的环境配置告诉WriteImpl，并且WriteThread访问的ExecutionStatus，XID和WriteBatch。WriteImpl会插入Prepare(xid)标记，然后是WriteBatch的内容，之后是EndPrepare()标记。不会发起memtable插入操作。当同一个事务对象发起提交，再一次，他调用到WriteImpl。这次，只有一个Commit()标记被插入到对应的WAL，并且WriteBatch的内容会被插入到对应的memtable。当对应事务的Rollback()被调用，事务的内容会被清理，并且如果事务已经就绪，调用WriteImpl，以插入一个Rollback(xid)标记。

这些所谓的'元标记'（Prepare(xid), EndPrepare(), Commit(xid), Rollback(xid)）不会直接插入到一个写批处理中。写流程（WriteImpl）会持有正在写入的事物的环境变量。它使用这个环境来插入相应的标记到WAl中（这样他们就被插入到完整的WriteBatch前面，中间不会有其他WriteBatch）。恢复的时候，这些标记会被MemTableInserter发现，他会使用这个来重新构造之前的准备好的事务。

事务时钟超时

目前，如果一个事务超时，这个事务提交有一个回调会失败。类似的，如果一个事务超时，那么他的锁就可以被其他事务偷取。这些机制在2PC中应该被保留 —— 差别是超时回调会在准备的时候被调用。如果事务在准备阶段没有超时，那么他不会再提交的时候超时。

TransactionDB修改

为了使用事务，用户必须打开一个TransactionDB。这个TransactionDB实例之后被用于构造Transaction。这个TransactionDB现在记录一个XID到所有已经创建的两阶段提交事务的映射。当一个事务被删除或者回滚，他从映射中被删除。同时有一个API用于查询所有准备好的事务。这个在MyRocks恢复的时候被使用。

TransactionDB同事还追踪一个所有包含准备段的日志号码的最小堆。当一个事务是'准备好'，他的WriteBatch会被写入一个日志，这个日志号会被存储在事务对象以及他的最小堆。当一个事务提交，他的日志号码会从小顶堆中删除，但是他不会被遗忘！现在需要memtable记录他需要的最老日志，直到他被落盘到L0。

# 写流程的修改

写流程可以被分解为两个主要关注的区域。DBImpl::WriteImpl(...)和MemTableInserter。多个客户端线程会调用到WriteImpl。第一个线程会被指定为leader，而一系列跟随的线程会被指定为'跟随者'。leader和一系列的跟随者会被聚在一起，变成一个逻辑组，指向一个'写组'。leader会处理该组的所有WriteBatches请求，把它们组合在一起，然后写出到WAl。根据写组的大小以及当前memtable是否愿意支持并行写入，leader可能会插入所有WriteBatches到memtable 或者 让每个线程分别插入他们的的WriteBatch到memtable。

所有memtable插入都是由MemTableInserter处理的。这是一个WriteBatch::Handler的实现 —— 一个WriteBatch迭代处理器。这个处理器遍历WriteBatch的所有元素（Put, Delete, Merge, 等待），并且对当前的MemTable执行对应的调用。MemTableInserter也会处理原地合并，删除和更新。

对写路径的修改会包括增加一个可选参数给DBImpl::WriteImpl。这个可选参数会是一个指针，指向写入数据的两阶段事务实例。这个对象会告诉写流程，当前两阶段事务的状态。一个2PC事务会在准备，提交，回滚分别调用一次WriteImpl —— 尽管提交和回滚都是互斥的操作。

```
Status DBImpl::WriteImpl(
  const WriteOptions& write_options, 
  WriteBatch* my_batch,
  WriteCallback* callback,
  Transaction* txn
) {
  WriteThread::Writer w;
  //...
  w.txn = txn; // writethreads also have txn context for memtable insert

  // we are now the group leader
  int total_count = 0;
  uint64_t total_byte_size = 0;
  for (auto writer : write_group) {
    if (writer->CheckCallback(this)) {
      if (writer->ShouldWriteToMem())
        total_count += WriteBatchInternal::Count(writer->batch)
       }
  }
  const SequenceNumber current_sequence = last_sequence + 1;
  last_sequence += total_count;

  // now we produce the WAL entry from our write group
  for (auto writer : write_group) {
    // currently only optimistic transactions use callbacks
    // and optimistic transaction do not support 2pc
   if (writer->CallbackFailed()) {
      continue;
    } else if (writer->IsCommitPhase()) {
      WriteBatchInternal::MarkCommit(merged_batch, writer->txn->XID_);
    } else if (writer->IsRollbackPhase()) {
      WriteBatchInternal::MarkRollback(merged_batch, writer->txn->XID_);
    } else if (writer->IsPreparePhase()) {
      WriteBatchInternal::MarkBeginPrepare(merged_batch, writer->txn->XID_);
      WriteBatchInternal::Append(merged_batch, writer->batch);
      WriteBatchInternal::MarkEndPrepare(merged_batch);
      writer->txn->log_number_ = logfile_number_;
    } else {
      assert(writer->ShouldWriteToMem());
      WriteBatchInternal::Append(merged_batch, writer->batch);
    }
  }
  //now do MemTable Inserts for WriteGroup
}
```

WriteBatchInternal::InsertInto可能会被修改为只迭代没有Transaction的写者或者COMMIT状态的Transaction。

写流程对MemTableInserter的修改

如你上面所见，当一个事务已经准备好，事务记录他准备段的日志号。在插入的时候，每个MemTable必须跟踪插入到他内部的准备段的最小日志号码。这个修改会发生在MemTableInserter里。我们会在日志声明周期部分讨论这个值如何使用。

# 恢复路径的修改

当前的恢复路径已经非常适合2PC了。他按时间顺序迭代所有日志里的所有批处理，然后根据日志号码，提供给MemTableInserter。MemTableInserter之后迭代每个批处理，然后把值插入到正确的MemTable。每个MemTable根据当前恢复中的日志编码，知道哪些值他可以忽略。

为了使恢复流程可以在2PC下工作，我们只需要修改MemTableInserter，让他能理解我们四个新的'元标记'。

记住：当一个2PC事务被提交，他包含多个列族的插入（多个memtable）。这些memtable会在不同时间落盘。我们仍旧使用CF日志号码来避免以恢复，两阶段，已提交的事务的重复插入。

考虑下列场景：


-	两阶段事务 TXN 插入到  CFA 和 CFB
-	TXN 在 LOG 1 准备好
-	TXN 在LOG 2标记为 COMMITTED
-	TXN 被插入到MemTables
-	CFA 落盘到 L0
-	CFA 的log_number 现在是 LOG 3
-	CFB 没有落盘，并且他仍旧指向LOG 1准备段
-	崩溃恢复
-	LOG 1 仍然存在，因为 CFB 在引用 LOG 1 准备段。
-	迭代从LOG 1开始的日志
-	CFB把准备好的数据插入memtable, 再次引用 LOG 1 的准备段
-	CFA 跳过LOG 2的提交标记的插入，因为他在LOG 3是一致的。
-	CFB 落盘到 L0 并且现在 LOG 3 也是一致的了。
-	LOG 1, LOG 2 可以被释放了。

重建事务

如前面所述，恢复路径的修改只要求修改MemTableInserter来处理新的元标记。因为在恢复的时候，我们不能访问一个完整的TransactionDB实例，我们必须凭空构造一个事务。这实质上是为所有恢复起来的准备好的事务构造一个XID->(WriteBatch,log_numb)的映射。当我们遇到一个Commit(xid)标记，我们尝试重新找到这个xid对应的事务，并且重新插入到Mem。如果我们遇到一个rollback(xid)标记，我们删除这个事务。在恢复的最后，我们只有一个包含所有准备好的事务的集合。之后我们通过这些对象构造完整的事务，获取需要的锁。RocksDB现在已经恢复到崩溃/关闭前的状态了。

日志生命周期

为了找出必须保留的最小日志，我们先找到每个列族的最小log_number_。

我们同时必须考虑在TransactionDB中已经准备好的段的堆的最小值。这代表了包含一个准备段但是没有提交的最早的日志。

我们同事必须考虑所有Memtable以及还没有落盘的ImmutableMemTables引用的准备段日志的最小值。

上面三个的最小值就是最早的还持有没有刷入L0的数据的日志。


