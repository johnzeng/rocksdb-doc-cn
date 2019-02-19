当前rocksdb中，默认情况下在写操作中任何一个错误（写WAL，Memtable落盘）都会导致db实例进入只读模式，之后的用户写操作都不会被接受。变量ErrorHandler::bg_error 会被设置到对应的失败Status。DBOptions::paranoid_checks选项控制Rocksdb在检查和确认错误的强度。默认值为true。下面的表展示不同场景的错误以及对db的潜在影响。

|错误原因|BG_ERROR_被设置的时机|
|---|---|
|同步 WAL (BackgroundErrorReason::kWriteCallback)|总是|
|Memtable插入失败 (BackgroundErrorReason::kMemTable)|	总是|
|Memtable落盘(BackgroundErrorReason::kFlush)|DBOptions::paranoid_checks为true|
|SstFileManager::IsMaxAllowedSpaceReached() 报告memtable落盘的时候已经达到最大空间(BackgroundErrorReason::kFlush)|总是|
|SstFileManager::IsMaxAllowedSpaceReached() 报告压缩期间到达最大空间 (BackgroundErrorReason::kCompaction)	|总是|
|DB::CompactFiles (BackgroundErrorReason::kCompaction)	|DBOptions::paranoid_checks为true|
|后台压缩 (BackgroundErrorReason:::kCompaction)|	DBOptions::paranoid_checks为true|
|写(BackgroundErrorReason::kWriteCallback)|DBOptions::paranoid_checks为 true|

# 探测

一旦数据库实例进入只读模式，下面的前端操作在所有后续调用中都会返回错误：

- DB::Write, DB::Put, DB::Delete, DB::SingleDelete, DB::DeleteRange, DB::Merge
- DB::IngestExternalFile
- DB::CompactFiles
- DB::CompactRange
- DB::Flush

返回的Status会包含具体的错误码，子码以及严重性。错误的严重性可以通过调用Status::severity()得到。有四个严重性等级：

- Status::Severity::kSoftError——这个严重性等级的错误，不会阻止DB的写入，但是他意味着db在一个降级模式。后台压缩可能没有按时执行。
- Status::Severity::kHardError——DB已经处于只读模式，但是一旦错误被修复，就可以恢复成读写模式
- Status::Severity::kFatalError——DB在只读模式。只有关闭DB，修复错误的根源，然后重新打开db，才有可能回复。
- Status::Severity::kUnrecoverableError——这是最高等级的严重性，并且意味着数据库将无法工作。可能可以关闭然后重新打开db，但是数据库的数据将不能保证正确。

除了上面提到的，在发生后台错误的时候，一个通知回调Status::Severity::kUnrecoverableError会被尽可能快地调用。

# 恢复

有两种办法从后台错误中恢复而不需要关闭数据库：

- EventListener::OnBackgroundError回调如果认为当前的错误不是那么严重，不需要停止后续的写错做，那么他可以修改错误状态。可以通过设置bg_error参数来完成这个工作。这么做可能会有风险，因为rocksdb可能无法保证DB的一致性。修改之前，检查BackgorundErrorReason以及错误的严重性。
- 调用DB::Resume()来恢复DB并且使之进入读写模式。这个方法会清理错误，清理所有废弃的文件，然后重启后台落盘以及压缩任务。目前，他只支持从压缩过程中产生的后台错误。未来，我们会加入更多的场景

