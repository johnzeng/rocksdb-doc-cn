# 简介

RocksDB支持一种通用消息日志基础设施。RocksDB能满足多种使用场景 —— 从低功耗移动系统到高端分布式服务器。这套框架帮助我们针对不同的需求拓展日志消息设施。相较于一个运行在服务器的严苛环境的应用，移动应用可能需要相对简单的日志机制。他还提供了将RocksDB日志消息集成到嵌入式应用的方法。

# 已有的日志系统

[Logger](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/env.h#L663) 类提供了一个接口定义了RocksDB的日志消息。几个Logger的实现如下：


实现 | 使用
------- | -------
NullLogger | 日志输出到/dev/null
StderrLogger | 把日志输出到std::err
HdfsLogger | 日志输出到HDFS
PosixLogger |  日志输出到POSIX文件系统
AutoRollLogger | 当文件到达一定大小后，自动翻滚。服务器的常用选择
WinLogger | 为Windows操作系统定制

# 自定义Logger

我们鼓励用户通过拓展已有的实现自己写日志系统。



