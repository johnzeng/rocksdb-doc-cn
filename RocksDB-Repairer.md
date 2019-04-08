# 概述

修复器允许在灾难之后， 在对一致性没有任何妥协的情况下，尽最大努力来修复尽可能多的数据。他不能保证把数据库恢复到一个抑制状态的时间点。

# 使用

注意CLI命令使用默认选项来修复你的DB，并且只增加在SST文件中找到的列族。如果你需要指定选项，比如，自定义的比较器，或者有根据列族不同的选项，或者希望声明额外的列族集合，你需要自己编写相关代码。

## 自行编写

为了自己编写代码，调用 include/rocksdb/db.h 中声明的其中一个RepairDB函数

## CLI

对于命令行使用方式，首先构建ldb，我们的管理员CLI工具：

```
$ make clean && make ldb
```

现在使用ldb的 repaire 子命令，指明你的DB。注意他打印info日志到stderr，所以你可能希望重定向他。这里我在一个./tmp目录下的DB运行它，在这里，我不小心删除了MANIFEST文件：

```
$ ./ldb repair --db=./tmp 2>./repair-log.txt
$ tail -2 ./repair-log.txt 
[WARN] [db/repair.cc:208] **** Repaired rocksdb ./tmp; recovered 1 files; 926bytes. Some data may have been lost. ****
Looks successful. MANIFEST file is back and DB is readable:
$ ls tmp/
000006.sst  CURRENT  IDENTITY  LOCK  LOG  LOG.old.1504116879407136  lost  MANIFEST-000001  MANIFEST-000003  OPTIONS-000005
$ ldb get a --db=./tmp
b
```

注意lost/目录。他持有所有可能在恢复过程中丢失的数据文件。

# 修复过程

修复过程分为4步：

- 查找文件
- 把日志转化为表
- 导出metadata
- 写描述符

## 查找文件

修复器遍历当前目录所有文件，然后根据他们的文件名进行分类。任何无法分类的文件都会被忽略

## 把日志转化为表

每个活跃的日志文件都会被重放。该文件中所有校验和不通过的节都会被跳过。我们有意保留一致的数据。

## 导出metadata

我们扫描所有的表，以计算：

- 该表最小/最大值
- 该表的最大序列号

如果我们无法扫描文件，我们忽略这个表

## 写描述符

我们生成描述符内容：

- 日志数量被设置为0
- 下一个文件编号被设置为 1 + 我们找到的最大文件编码
- 最后序列号 被设置为在所有表中找到的最大的序列号
- 压缩(compaction)指针被清理
- 每个表文件都会加入到level0

## 可能的优化

- 指定总大小，然后选择最接近的最大层M
- 根据表的最大序列号对表排序
- 对于每个表：如果他跟前面的表有交集，放在level 0，否则，放在level M
- 我们可以提供选项来指定时间一致性恢复和不安全恢复（可能的话，忽略文件的校验和错误）
- 存储每个标的元数据（最小，最大，最大序列号。。。）到标的元段以加快ScanTable

