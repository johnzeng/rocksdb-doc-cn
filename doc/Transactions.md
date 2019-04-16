当使用TransactionDB或者OptimisticTransactionDB的时候，RocksDB将支持事务。事务带有简单的BEGIN/COMMIT/ROLLBACK API，并且允许应用并发地修改数据，具体的冲突检查，由Rocksdb来处理。RocksDB支持悲观和乐观的并发控制。

注意，当通过WriteBatch写入多个key的时候，RocksDB提供原子化操作。事务提供了一个方法，来保证他们只会在没有冲突的时候被提交。于WriteBatch类似，只有当一个事务提交，其他线程才能看到被修改的内容（读committed）。

# TransactionDB

当使用TransactionDB的时候，所有正在修改的RocksDB里的key都会被上锁，让RocksDB执行冲突检测。如果一个key没法上锁，操作会返回一个错误。当事务被提交，数据库保证这个事务是可以写入的。

一个TransactionDB在由大量并发工作压力的时候，相比OptimisticTransactionDB有更好的表现。然而，由于非常过激的上锁策略，使用TransactionDB会有一定的性能损耗。TransactionDB会在所有写操作的时候做冲突检查，包括不使用事务写入的时候。

上锁超时和限制可以通过TransactionDBOptions进行调优。

```cpp
TransactionDB* txn_db;
Status s = TransactionDB::Open(options, path, &txn_db);

Transaction* txn = txn_db->BeginTransaction(write_options, txn_options);
s = txn->Put(“key”, “value”);
s = txn->Delete(“key2”);
s = txn->Merge(“key3”, “value”);
s = txn->Commit();
delete txn;
```

默认的写策略是WriteCommitted。还可以选择WritePrepared 和WriteUnprepared。更多内容请参考[这里]()

# OptimisticTransactionDB

乐观事务提供轻量级的乐观并发控制给那些多个事务间不会有有高的竞争／干涉的工作场景。

乐观事务在预备写的时候不使用任何锁。作为替代，他们把这个操作推迟到在提交的时候检查，是否有其他人修改了正在进行的事务。如果和另一个写入有冲突（或者他无法做决定），提交会返回错误，并且没有任何key都不会被写入。

乐观的并发控制在处理那些偶尔出现的写冲突非常有效。然而，对于那些大量事务对同一个key写入导致写冲突频繁发生的场景，却不是一个好主意。对于这些场景，使用TransactionDB是更好的选择。OptimisticTransactionDB在大量非事务写入，而少量事务写入的场景，会比TransactionDB性能更好


```cpp
DB* db;
OptimisticTransactionDB* txn_db;

Status s = OptimisticTransactionDB::Open(options, path, &txn_db);
db = txn_db->GetBaseDB();

OptimisticTransaction* txn = txn_db->BeginTransaction(write_options, txn_options);
txn->Put(“key”, “value”);
txn->Delete(“key2”);
txn->Merge(“key3”, “value”);
s = txn->Commit();
delete txn;
```

# 从一个事务中读取

事务对当前事务中已经批量修改，但是还没有提交的key提供简单的读取操作。

```cpp
db->Put(write_options, “a”, “old”);
db->Put(write_options, “b”, “old”);
txn->Put(“a”, “new”);

vector<string> values;
vector<Status> results = txn->MultiGet(read_options, {“a”, “b”}, &values);
//  The value returned for key “a” will be “new” since it was written by this transaction.
//  The value returned for key “b” will be “old” since it is unchanged in this transaction.
```
使用Transaction::GetIterator()，你还可遍历那些已经存在db的键以及当前事务的键。

# 设定一个快照

默认的，事务冲突检测会校验没有其他人在*事务第一次修改这个key之后*，对这个key做了修改。这个解决方案在多数场景都是足够的。然而，你可能还希望保证没有其他人在*事务开始之后*，对这个key做了修改。可以通过创建事务后调用SetSnapshot来实现。

默认行为：

```cpp
// Create a txn using either a TransactionDB or OptimisticTransactionDB
txn = txn_db->BeginTransaction(write_options);

// Write to key1 OUTSIDE of the transaction
db->Put(write_options, “key1”, “value0”);

// Write to key1 IN transaction
s = txn->Put(“key1”, “value1”);
s = txn->Commit();
// There is no conflict since the write to key1 outside of the transaction happened before it was written in this transaction.
```

使用SetSnapshot：

```cpp
txn = txn_db->BeginTransaction(write_options);
txn->SetSnapshot();

// Write to key1 OUTSIDE of the transaction
db->Put(write_options, “key1”, “value0”);

// Write to key1 IN transaction
s = txn->Put(“key1”, “value1”);
s = txn->Commit();
// Transaction will NOT commit since key1 was written outside of this transaction after SetSnapshot() was called (even though this write
// occurred before this key was written in this transaction).

```

注意，在前一个例子，如果这是一个TransactionDB，Put会失败。如果是OptimisticTransactionDB，Commit会失败。

# 可重复读

于普通的RocksDB读相似，有可以在ReadOptions指定一个Snapshot来保证事务中的读是可重复读。

```cpp
read_options.snapshot = db->GetSnapshot();
s = txn->GetForUpdate(read_options, “key1”, &value);
…
s = txn->GetForUpdate(read_options, “key1”, &value);
db->ReleaseSnapshot(read_options.snapshot);
```

注意，在ReadOptions设定一个快照只会影响读出来的数据的版本。他不会影响事务是否可以被提交。

如果你已经调用了SetSnapshot，你也可以使用在事务里设定的同一个快照。

```cpp
read_options.snapshot = txn->GetSnapshot();
Status s = txn->GetForUpdate(read_options, “key1”, &value);
```

# 读写冲突保护

GetForUpdate会保证没有其他写入者会修改任何被这个事务读出的key。

```cpp
// Start a transaction 
txn = txn_db->BeginTransaction(write_options);

// Read key1 in this transaction
Status s = txn->GetForUpdate(read_options, “key1”, &value);

// Write to key1 OUTSIDE of the transaction
s = db->Put(write_options, “key1”, “value0”);
```
如果这个事务是通过TransactionDB创建，Put操作要么超时，要没就会被阻塞直到事务被提交或者放弃。如果这个事务通过OptimisticTransactionDB创建，那么Put会成功，但是事务会在调用txn->Commit()的时候失败。

```cpp
// Repeat the previous example but just do a Get() instead of a GetForUpdate()
txn = txn_db->BeginTransaction(write_options);

// Read key1 in this transaction
Status s = txn->Get(read_options, “key1”, &value);

// Write to key1 OUTSIDE of the transaction
s = db->Put(write_options, “key1”, “value0”);

// No conflict since transactions only do conflict checking for keys read using GetForUpdate().
s = txn->Commit();
```

# 调优／内存使用

在内部，事务需要追踪那些key最近被修改过。现有的内存写buffer会因此被重用。当决定内存中保留多少写buffer的时候，事务仍旧遵从已有的max_write_buffer_number选项。另外，使用事务不影响落盘和压缩。

可能切换到使用[乐观]事务DB使用更多的内存。如果你曾经给max_write_buffer_number设置一个非常大的值，一个标准的RocksDB实例永远都不会逼近这个最大内存限制。然而，一个[乐观]事务DB会尝试使用尽可能多的写buffer。这个可以通过减小max_write_buffer_number或者设置max_write_buffer_number_to_maintain为一个小于max_write_buffer_number的值来进行调优。

OptimisticTransactionDB：在提交的时候，乐观事务会使用内存写buffer来做冲突检测。为此，缓存的数据必须比事务中修改的内容旧。否则，Commit会失败。增加max_write_buffer_number_to_maintain以减小由于缓冲区不足导致的提交失败。

TransactionDB：如果使用了SetSnapshot，Put/Delete/Merge/GetForUpdate操作会先检查内存的缓冲区来做冲突检测。如果没有足够的历史数据在缓冲区，那么会检查SST文件。增加max_write_buffer_number_to_maintain会减少冲突检测过程中的SST文件的读操作。

# 保存点

出了Rollback，事务还可以通过SavePoint来进行部分回滚。

```cpp
s = txn->Put("A", "a");
txn->SetSavePoint();
s = txn->Put("B", "b");
txn->RollbackToSavePoint()
s = txn->Commit()
// Since RollbackToSavePoint() was called, this transaction will only write key A and not write key B.
```

# 底层实现

一个高度简明的概括，展示了事务的工作原理。

## 读快照

每一个RocksDB的更新都是通过插入一个带有强制自增的序列号的项来实现的。给即将要被（事务或者非事务）DB用来创建快照的read_options.snapshot赋值一个序列号，可以只读到小于这个序列号的内容。例如，读快照→ DBImpl::GetImpl

除此之外，事务还可以调用TransactionBaseImpl::SetSnapshot，这个接口会调用DBImpl::GetSnapshot。他实现了两个目标：

- 返回当前的序列号：事务会使用这个序列号（而不是他的写入值的序列号）来检测写－写冲突 → TransactionImpl::TryLock → TransactionImpl::ValidateSnapshot → TransactionUtil::CheckKeyForConflicts
- 确保小于这个序号的值不会被压缩删除。例如 (snapshots_.GetAll)。这些快照必须被调用者释放（DBImpl::ReleaseSnapshot）

## 读写冲突检测

读写冲突可以通过升级为写写冲突来防止：通过GetForUpdate（而不是Get）来做读操作。

## 写写冲突检测：悲观方式

写写冲突在写入的时候用一个锁表来进行检测。

非事务更新（put，merge，delete）在内部其实是以一个事务来运行的。所以每个更新都是通过事务→ TransactionDBImpl::Put

每个更新都会先申请一个锁→ TransactionImpl::TryLock 
TransactionLockMgr::TryLock在每个列族只有16个锁→ size_t num_stripes = 16

Commit只是简单地把批量写写到WAL然后通过调用DBImpl::Write来写入Memtable→ TransactionImpl::Commit

为了支持分布式事务，客户端可以在写之后先调用Prepare。他会把数据写入WAL，但是不会写入Memtable，这允许机器崩溃之后恢复→ TransactionImpl::Prepare 如果Prepare被调用，Commit会写一个提交记录到WAL然后把数值写入MemTable。这是通过在批量写增加值之前调用MarkWalTerminationPoint来实现的。

## 写写冲突检测：乐观方式

写写冲突会在提交的时候检查其最后一个序列号，来检测冲突。

每次更新把key加入到一个内存的vector中→ TransactionDBImpl::Put 与 OptimisticTransactionImpl::TryLock

Commit把OptimisticTransactionImpl::CheckTransactionForConflicts作为回调连接到批量写→ OptimisticTransactionImpl::Commit，他会被DBImpl::WriteImpl通过写入进行回调->CheckCallback

冲突检测的逻辑在TransactionUtil::CheckKeysForConflicts中实现

- 只检测在内存中出现的key的冲突和失败。
- 冲突检测是通过比对每个key的最后的序列号（DBImpl::GetLatestSequenceForKey）与用于写入的序列号来实现的。



