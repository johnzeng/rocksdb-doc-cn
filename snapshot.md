一个快照会捕获在创建的时间点的DB的一致性视图。快照在DB重启之后将消失。

# API 使用

- 通过GetSnapshot API创建一个快照
- 通过设置ReadOptions::snapshot来读取快照的内容
- 当读取结束，调用ReleaseSnapshot释放相关资源

# 实现

## Flush／compaction Representation

一个快照相当于一个SnapshotImpl类的小型对象。他只持有部分简单的字段，比如快照生成的时候的seqnum。

Snapshot会存储在一个DBImpl持有的链表里。其中一个好处是，我们可以在获取DB互斥锁钱分配好这个链表的节点。然后在持有互斥锁的时候，我们只需要更新链表指针。更进一步，ReleaseSnapshot可以对所有的快照以任意顺序被调用。使用链表，我们不需要移动所有节点就可以删除一个节点了。

## 伸缩性

使用链表的唯一问题是，尽管他是顺序排列的，他还是不可以使用二分查找。在flush/comapction的时候，如果我们需要找到一个key在最早的哪个snapshot中是可见的，我们必须煮个扫描快照链表。当许多快照存在的时候，这个扫描会非常明显的拖慢flush/compaction到一个写失速的点。我们在有成百上千个快照的时候就观察到了这种问题。

