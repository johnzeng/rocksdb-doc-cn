# 概述

我们有能力使一个RocksDB实例，或者多个实例聚合在一起， 的磁盘使用量，保持在一定数值以下。在一些一个文件系统被多个应用使用，并且其他应用系统需要与预先增长的数据库隔离的情况，是非常有用的。

追踪硬盘空间使用以及，可选的，限制数据库大小，是通过rocksdb::SstFileManager来实现的。他通过调用NewSstFileManager来生成，返回的对象被赋值给DBOptions::sst_file_manager。可以在多个DB实例之间共享一个SstFileManager。

# 使用

## 追踪db大小

通过调用SstFileManager::GetTotalSize()，调用者可以知道DB使用的总磁盘空间。他返回所有SST文件使用的总大小。WAL文件不会被包括进去。如果同一个SstFileManager被用于多个DB实例，GetTotalSize会返回所有实例的总大小。

## 限制DB大小

通过调用SstFileManager::SetMaxAllowedSpaceUsage()以及，可选的，SstFileManager::SetCompactionBufferSize()，可以限制磁盘空间使用的方式。两个函数都接受一个参数，该参数以byte为单位，声明希望使用的尺寸。前者设置一个硬性DB大小限制，后者声明在决定是否压缩前，内部需要保留的空间。

设置最大的DB大小可以从下面几方面影响DB行为。

- 每当一个新的SST文件通过落盘或者压缩被创建，SstFileManager::OnAddFile() 都会被调用，用于更新总共被使用的大小。如果这回导致总大小大于限制，ErrorHandler::bg_error_ variable会被设置到Status::SpaceLimit()，并且创建该SST文件的DB实例会进入只读模式。更多信息，参考[后台错误处理]()
- 开始一个压缩前，RocksDB会检查是否有足够的空间用来创建输出的SST文件。这是通过调用SstFileManager::EnoughRoomForCompaction()来完成的。这个函数用一种保守的方式估计输出的大小为所有输入的SST文件的总大小。如果设置有压缩buffer，还会加上这个buffer，得到的输出结果会大于SstFileManager::SetMaxAllowedSpaceUsage()设置的值，压缩不会被允许执行。压缩线程会休眠1秒之后，把列族重新加入到压缩队列。所以说这是非常有用的压缩比率阈值。

