# 介绍

很长一段时间，硬盘的意思都是数据存储的持久化介质。随着SSD的引入，我们现在有一个比传统磁盘快非常多的持久化介质，虽然他的擦写次数和容量会更小，是的我们可以有机会探索tiered存储架构。开源实现，如使用SSD闪盘缓存以及供服务器应用的，使用更好的disk作为tiered存储。RocksDB Persistent读缓存是一个尝试使用tiered存储架构来应对未知设备和操作系统的RocksdB生态系统。

# Tiered存储 VS Tiered缓存

RocksDB用户可以通过tiered存储部署的方式，或者使用tiered缓存部署的方式 从tiered存储架构获得好处。使用tiered存储方式，你可以分散LSM的多层持久化存储的内容。使用tiered缓存方式，用户可以使用更快的持久化介质作为读缓存，以服务于LSM的高频访问部分，并且增强RocksDB的性能。

Tiered缓存在数据便携性方面有一些优势，因为缓存是性能的一个增强。存储部分可以在没有缓存的情况下继续工作。

# 关键功能

## 硬件无关

这个持久化读缓存是一个通用实现，他不是对任何特定设备设计的。与针对不同的硬件进行设计不同，我们给用户留了一个机制，用来描述访问设备的最佳方式，并且IO方式也可以针对不同的描述进行配置。

写代码流程可以使用下面的公式描述：

```
{ Block Size, Queue depth, Access/Caching Technique }
```

读代码流程可以通过下列公式描述：

```
{ Access/Caching Technique }
```

块大小(Block Size)描述了读/写的大小。在SSD，这通常是擦出块大小。

队列深度(Queue depth)对应于设备能展现最优性能的地方。

访问/缓存技术(Access/Caching Technique)别用于描述访问设备的最佳方式。举个例子，使用直接IO，就适合特定设备/应用，而带缓冲的方式则适合其他。

## OS无关

持久化读缓存使用RocksDB抽象来构建，并且支持所有RocksDB支持的平台。

## 可插拔式

由于这是一个缓存实现，缓存可以，也可能不可以在重启的时候可用。

# 设计以及实现细节

持久化读缓存的实现有三个基本原件。

## 块查找索引

这是一个可伸缩的内存哈希索引，把一个给定的LSM块地址映射到一个缓存记录定位器。缓存记录定位器帮助定位块数据在缓存的位置。缓存记录可以被描述为{file-id, offset, size }。

## 文件查找索引/LRU

这是一个可伸缩的内存哈希索引，允许基于LRU进行淘汰。这个索引把一个给定的文件描述符映射到他的引用对象的抽象。这个对象抽象可以被用于从缓存读取数据。当我们持久化缓存的空间不够的时候，我们淘汰最近最不常用的项。

## 文件分布

缓存以一系列文件的形式存储在文件系统。每个文件包含一系列的记录，这些记录包含对应RocksDB的LSM块的数据。

# API

请参考下属链接找到公开API。

[https://github.com/facebook/rocksdb/blob/master/include/rocksdb/persistent_cache.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/persistent_cache.h)



