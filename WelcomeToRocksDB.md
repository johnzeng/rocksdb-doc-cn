# 欢迎使用RocksDB
RocksDB是使用C++编写的嵌入式kv存储引擎，其键值均允许使用二进制流。由Facebook基于levelDB开发， 提供向后兼容的levelDB API。

RocksDB针对Flash存储进行优化，延迟极小。RocksDB使用LSM存储引擎，纯C++编写。Java版本RocksJava正在开发中。参见[RocksJavaBasic](https://rocksdb.org.cn/doc/RocksJava-Basics.html)。

RocksDB依靠大量灵活的配置，使之能针对不同的生产环境进行调优，包括直接使用内存，使用Flash，使用硬盘或者HDFS。支持使用不同的压缩算法，并且有一套完整的工具供生产和调试使用。

## 功能
- 为需要存储TB级别数据到本地FLASH或者RAM的应用服务器设计
- 针对存储在高速设备的中小键值进行优化——你可以存储在flash或者直接存储在内存
- 性能碎CPU数量线性提升，对多核系统友好

## LevelDB所没有的功能
RocksDB增加了许多新功能，参考[features not in LevelDB](https://rocksdb.org.cn/doc/Features-Not-in-LevelDB.html)

## 开始
参考左边的菜单获得完整的内容表单。多数读者希望从开发者指南的 [概述]()和[基本操作]()开始。可以跟随[安装选项以及基础调优]()进行第一次安装设置。也可以看看[FAQ]()。还有一份[调优指南]() 给高级用户。

## 报告Bug或者寻求帮助
如果你有任何疑问，请着着这个[指南]()进行BUG上报或者寻求帮助

## BLOG
这个是我们的博客 [blog](rocksdb.org/blog)

## 项目历史

-[RocksDB项目历史](http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html)
-[深入了解RocksDB](https://www.facebook.com/notes/facebook-engineering/under-the-hood-building-and-open-sourcing-rocksdb/10151822347683920)

##链接
- [样例程序](https://github.com/facebook/rocksdb/tree/master/examples)
- [官方博客](http://rocksdb.org/blog/)
- [stack overflow:rocksdb](https://stackoverflow.com/questions/tagged/rocksdb)
- [演讲](https://rocksdb.org.cn/doc/Talks.html)

## 联系我们
- [开发者讨论组]()

文档License[here]()


