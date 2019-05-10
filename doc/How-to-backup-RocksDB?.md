# Backup API

对于C++ APi，参考`include/rocksdb/utilities/backupable_db.h`。主要的抽象是备份引擎，会暴露一些简单的接口用于创建备份，获取备份信息，以及从备份中恢复。有两个不同的备份引擎实现:(1)`BackupEngine`用于创建新的备份，以及(2)`BackupEngineReadOnly`用于从备份恢复数据。他们都可以用于获取备份相关的信息。

# 创建并校验一份备份

在RocksDB，我们实现了一个简单的方法来备份db，以及校验其正确性。这里是一个简单的例子：

```cpp
 #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    #include <vector>

    using namespace rocksdb;

    int main() {
        Options options;                                                                                  
        options.create_if_missing = true;                                                                 
        DB* db;
        Status s = DB::Open(options, "/tmp/rocksdb", &db);
        assert(s.ok());
        db->Put(...); // do your thing

        BackupEngine* backup_engine;
        s = BackupEngine::Open(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"), &backup_engine);
        assert(s.ok());
        s = backup_engine->CreateNewBackup(db);
        assert(s.ok());
        db->Put(...); // make some more changes
        s = backup_engine->CreateNewBackup(db);
        assert(s.ok());

        std::vector<BackupInfo> backup_info;
        backup_engine->GetBackupInfo(&backup_info);

        // you can get IDs from backup_info if there are more than two
        s = backup_engine->VerifyBackup(1 /* ID */);
        assert(s.ok());
        s = backup_engine->VerifyBackup(2 /* ID */);
        assert(s.ok());
        delete db;
        delete backup_engine;
    }
```

这个简单的例子会创建一对备份到`/tmp/rocksdb_backup`。注意你可以在同一个引擎里创建并校验多个备份。

备份正常来说是增量的（参考`BackupableDBOptions::share_table_files`）。你可以使用`BackupEngine::CreateNewBackup()`创建一个新的备份，并且只有最新的数据会被拷贝到备份目录（更多细节，参考[底层实现](#底层实现)）

一旦你有一些保存好的备份，你可以调用`BackupEngine::GetBackupInfo()`来获取一个备份列表以及时间戳对应的备份信息大小信息（注意，所有备份的和大于实际的备份目录的大小，因为有些数据会被多个备份共享）。备份可以通过自增ID做唯一标志。

当`BackupEngine::VerifyBackups()`被调用，他会检查备份目录的文件大小和db目录下原始文件的大小。然而，我们不检查校验和，因为这需要读取所有数据。注意`BackupEngine::VerifyBackups()`唯一的合理用途是 同一个引擎在因为在备份过程中，状态被捕获，被用于创建（多个）备份后，在一个备份引擎上调用他。

# 从备份恢复

恢复备份也很简单：

```cpp
    #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    using namespace rocksdb;

    int main() {
        BackupEngineReadOnly* backup_engine;
        Status s = BackupEngineReadOnly::Open(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"), &backup_engine);
        assert(s.ok());
        backup_engine->RestoreDBFromBackup(1, "/tmp/rocksdb", "/tmp/rocksdb");
        delete backup_engine;
    }
```

这段代码会读取/tmp/rocksdb里的第一个备份。`BackupEngineReadOnly::RestoreDBFromBackup()`的第一个参数是备份ID，第二个参数是目标DB目录，第三个是日志文件的目标地址（在某些DB，他们可能跟DB目录不同，但是大多数时候他们是同一个目录，参考Options::wal_dir了解更多信息）。`BackupEngineReadOnly::RestoreDBFromLatestBackup()`会从最新的备份里恢复DB，也就是说，ID最高的那个备份。

每个恢复的文件的校验和都要被计算，然后与备份的文件进行比较。如果校验和不一致，恢复过程会终止，并且会返回`Status::Corruption`

你必须重新打开任何线上的数据库，才能看到恢复的数据。

# 备份目录结构

```
/tmp/rocksdb_backup/
├── LATEST_BACKUP
├── meta
│   └── 1
├── private
│   └── 1
│       ├── CURRENT
│       ├── MANIFEST-000008
|       └── OPTIONS-000009
└── shared_checksum
    └── 000007_1498774076_590.sst
```

`LATEST_BACKUP`是一个包含最高备份ID的文件。在我们上面的例子里，它里面有一个"1"。他被用于获取最新的备份编号，但是现在不再需要他了，因为有一个更简单的通过META文件获取的方式。这个文件会在Rocksdb5.0被移除。

`meta` 目录包含一个“meta文件”，描述每一个备份，他的名字为备份ID。例如，一个meta文件包含一个包含该备份的所有文件的列表。格式在其实现文件中被描述（`utilities/backupable/backupable_db.cc`）。

`private`目录总是包含非SST文件（options, current, manifest, 以及WAL）。如果`Options::share_table_files`被置空，他还会包含SST文件。

`shared`目录（未展示）在设置了`Options::share_table_files`而没有设置`Options::share_files_with_checksum`的时候包含SST文件。在这个目录，文件直接用他们在原DB里的名称命名。所以他只应该被用于备份一个单一RocksDB实例；否则，文件名会冲突。

`shared_checksum`目录在置了`Options::share_table_files`和`Options::share_files_with_checksum`的时候包含SST文件。在这个目录，文件用他们在原DB里的名称，大小和校验和命名。这些属性能帮助来自多个RocksDB实例的文件获得唯一标志。

# 备份性能

请注意，备份引擎的`Open()`的时间开销 跟已经存在的备份数量成正比，因为我们需要初始化已经存在的每个备份的文件信息。所以如果你面向一个远程文件系统（如HDFS），而你可能有很多备份，那么初始化备份引擎可能需要花点时间，因为有网络交互的时间。我们推荐你保持备份引擎存活，并且不要每次都重新创建。

另一个加快备份引擎初始化的方法是删除不用的备份。为了删除不用的备份，只需要调用`PurgeOldBackups(N)`，N是你希望保留的备份数量。除了最新的N份备份，所有备份都会被清理。你可以通通过调用`DeleteBackup(id)`来删除任意备份。

同时注意性能也受从本地db读取然后拷贝到备份这个过程的影响。由于你可能使用不同的环境来读取和拷贝，并行瓶颈可能在任意一边出现。例如，如果本地db是HDD，使用更多的线程来备份（参考[高级使用](#高级使用)）不会有效果，因为这个场景的瓶颈是磁盘度性能，可能已经饱和了。同事，一个很小的HDFS集群可能无法获得好的并发性。如果本地db在SSD而备份目标在一个大容量HDFS，就比较好了。在我们的压测下，使用16线程会减小备份时间到单线程的1/3。

# 底层实现

当你调用`BackupEngine::CreateNewBackup()`，他执行一下动作：

1. 关闭文件删除
2. 获取存活文件（包括表文件，current，选线和manifest文件）
3. 拷贝存货文件到备份目录。由于表文件是不可修改，并且文件名唯一，如果备份目录下已经有这个文件了，我们就不拷贝他了。例如，如果有一个文件叫`00050.sst`在备份目录里，然后`GetLiveFiles()`返回了`00050.sst`，我们不会拷贝这个文件到备份目录。然而，不管文件是否被拷贝，所有文件的校验和都需要被计算。如果一个文件已经存在，算出来的校验和会跟之前算出来的校验和做比较，以确保没有意料外的事情发生。如果校验和不一致，备份会终止，系统恢复到`BackupEngine::CreateNewBackup()`调用前的状态。值得注意的是，一个备份终止可能意味着当前db中的文件出错，也可能是备份的文件出错。另外，manifest和current文件总是被拷贝到private目录，因为他们不是不可变得。
4. 如果`flush_before_backup`为false，我们还需要把WAL拷贝到备份目录。我们调用`GetSortedWalFiles()`然后拷贝所有存活文件到备份目录。
5. 重新打开文件删除。

# 高级使用

我们可以恢复用户定义的元数据备份。把你的元数据传给`BackupEngine::CreateNewBackupWithMetadata()`然后后续通过`BackupEngine::GetBackupInfo()`读取。例如，这个可以使用不同于我们的自增id的自定义id，以用于区分备份。

我们现在也备份和恢复option文件了。回复后，你可以从db目录使用`rocksdb::LoadLatestOptions()`或者`rocksdb:: LoadOptionsFromFile()`加载option。限制是，并不是options里的所有内容都可以转换成一个文件中的text。加载并恢复后，你仍旧需要一些步骤来手工设置一些没有设置的信息。好消息是，你比以前需要做的事情少了许多。

你需要初始化一些env，然后给backup_target初始化`BackupableDBOptions::backup_env`。把你的备份根目录写入`BackupableDBOptions::backup_dir`在该目录下，文件会按照上述说明组织。

`BackupableDBOptions::share_table_files`控制备份是否增量完成。如果为真，SST文件会去到"shared"目录。如果不同的SST文件使用同一个名称，就会造成冲突（比如说，如果多个数据库使用同一个备份目标文件夹）。

`BackupableDBOptions::share_files_with_checksum`控制共享文件如何被区分。如果为真，共享SST文件会通过校验和，大小，以及序列号进行区分。这可以防止上述多个数据库使用同一个目录带来的冲突。

`BackupableDBOptions::max_background_operations`控制备份和恢复的时候用于拷贝文件的线程数。对于分布式文件系统，如HDFS，增加拷贝并发数可以获得很好的收益。

`BackupableDBOptions::info_log`是一个Logger对象，非空时，用于打印LOG日志。参考[Logger](Logger.md)

如果`BackupableDBOptions::sync`为真，每次文件写，我们都会使用`fsync(2)`来同步文件数据和元数据到磁盘上，保证重启或者机器崩溃后的备份持久化。设置为false会加快一点速度，但是一些（更新的）备份可能不能持久化。尽管在大多数情况下，一切都相安无事。

如果你设置`BackupableDBOptions::destroy_old_data`为true，创建新的`BackupEngin`会删除目标备份目录下的所有旧的备份。

`BackupEngine::CreateNewBackup()`方法需要一个参数`flush_before_backup`，默认为false。如果`flush_before_backup`为true，`BackupEngine`会先发起一个memtable落盘，之后才开始拷贝DB文件到备份目录。这样会不再拷贝WAL文件到备份目录（因为落盘后会删除他们）。如果`flush_before_backup`为false，备份不会发起落盘。这时，备份会包括memtable对应的WAL文件。不管`flush_before_backup`为何，备份总是与当前状态一致。

# 进一步阅读

对于具体实现，参考`utilities/backupable/backupable_db.cc`



