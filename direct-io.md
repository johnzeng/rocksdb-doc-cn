# 介绍

直接IO是一个系统层的功能，支持用户层直接读/写存储设备而不使用系统的页缓存。缓冲式IO通常是大多数操作系统的默认IO模式。

## 为什么我们需要它

使用缓冲式IO的时候，数据会在存储介质和内存间被拷贝两次，因为页缓存式两者的代理。在多数情况，使用页缓存可以获得更好的性能。但是对于自缓存应用，例如RocksDB，应用自身会比OS更好地了解逻辑和数据语意，这就使得应用可以通过他们对数据的了解，更好地以应用定义的数据块为单位，实现更有效的缓存替换算法。另一边，在某些情况，我们希望某些数据不使用系统缓存。这时候，直接IO会是一个更好的选择。

## 实现

打开直接IO的方式与OS以及文件系统对直接IO的支持有关。在使用这个功能之前，请检查文件系统是否支持直接IO。RocksDB已经处理了这些系统依赖的兼容性问题，但是我们想分享一下实现细节。

	打开文件
		对于LINUX，需要加入O_DIRECT标记。对于Mac OSX，没有O_DIRECT。作为替换，fcntl(fd, F_NOCACHE, 1)，fd是文件描述符，看起来是权威方案。对于Windows，有一个叫做FILE_FLAG_NO_BUFFERING的标记，是Windows中O_DIRECT的替代品。
	
	文件读写
		直接IO要求文件读写是对齐的，这意味着，位置游标（偏移），#bytes以及buffer地址，必须与底层存储的逻辑扇区大小对齐。这样，位置游标应该，缓冲区地址必须，与逻辑扇区的大小边界对齐，读取和写入的数据量必须是逻辑扇区的大小的整数倍。RocksDB在FileReade和FileWrite里面实现了所有的对齐逻辑，一个基于File类的更高的抽象层，用来处理OS不关心对齐问题。因此，不同的OS可能有他们自己实现的File类。
		
		
# API

由于options.h提供的两个新的API，使用直接IO非常简单：

```cpp
  // Use O_DIRECT for user reads
  // Default: false
  // Not supported in ROCKSDB_LITE mode!
  bool use_direct_reads = false;

  // Use O_DIRECT for both reads and writes in background flush and compactions
  // When true, we also force new_table_reader_for_compaction_inputs to true.
  // Default: false
  // Not supported in ROCKSDB_LITE mode!
  bool use_direct_io_for_flush_and_compaction = false;
```
代码是自解释的。

你可能还需要其他选项来优化直接IO的性能：

```cpp
// options.h
// Option to enable readahead in compaction
// If not set, it will be set to 2MB internally
size_t compaction_readahead_size = 2 * 1024 * 1024; // recommend at least 2MB
// Option to tune write buffer for direct writes
size_t writable_file_max_buffer_size = 1024 * 1024; // 1MB by default
// DEPRECATED!
// table.h
// If true, block will not be explicitly flushed to disk during building
// a SstTable. Instead, buffer in WritableFileWriter will take
// care of the flushing when it is full.
// This option is deprecated and always be true
bbto.skip_table_builder_flush = true;

```

最近的版本如果开启了直接IO，会把这些选项自动设置好。

## 注意

allow_mmap_reads不可以和use_direct_reads或者use_direct_io_for_flush_and_compaction一起用。allow_mmap_write不能和use_direct_io_for_flush_and_compaction一起用。也就是，他们不能同时为true。

use_direct_io_for_flush_and_compaction和use_direct_reads只会对SST文件IO生效，WAL的IO或者MANIFEST的IO不会生效。对WAL和Manifest文件直接IO目前还不支持。

打开直接IO后，压缩写将不再写入OS页缓存，所以第一次读会从真实IO读。某些用户可能知道RocksDB有一个功能叫做压缩块缓存，用于在直接IO的时候替换页缓存。但是打开前请阅读下面的备注：

- 碎片化：RocksDB的压缩块不是根据页大小对齐的。一个压缩块存放在一个malloc生成的RocksDB的压缩块缓存中。这通常意味着内存使用的碎片化。OS的页缓存会好点，因为他缓存整个物理页。如果某些连续的块总是热数据，OS的页缓存会使用更少的内存来缓存他们。
- OS页缓存提供预读取。在RocksDB这个是默认关闭的，但是用户可以选择开启他们。这在范围扫描的情况下会很好用。RocksDB压缩缓存在这种情况没有任何对应的功能。
- 可能存在bug。RocksDB的压缩块缓存以前从来没有被用过。我们确实看到有外部用户给它报告bug了，但是我们不会在这个模块进行更多改进。

