# 关于压缩（Compaction）与压缩（Compression）的区别

**这段内容在原文是没有的，翻译君实在没有什么好办法，不得不加上这段话**

RocksDB涉及两个压缩概念，英文原文是Compaction和Compression。两个术语用中文翻译，都是“压缩”，实际上大家交流的时候也都是使用“压缩”。这个wiki很良心地写了两个压缩的内容，用英语能区分，但是用中文。。。嗯。。。。这里简单介绍下两个压缩的区别。

compaction，在RocksDB，或者说LSM存储中，指的是把数据从Ln层，存储到Ln+1层这个过程，例如把重复的旧的数据删除之类的。

compression，在这里指的是数据压缩，把1MB的数据压缩成500KB这样。数据还是那些数据，只是从明文，变成了压缩之后的数据。

他们的关系大概是：

通过Compaction把数据压缩到不同的层，每层使用不同的Compression算法压缩数据，减少存储空间。

下面开始是正文。

--

压缩算法限制LSM树的形状。他们决定了那些排序结果可以被合并以及那些排序结果需要被一个读取操作访问。你可以参考[多线程压缩]()了解关于RocksDB压缩的更多细节。

# 压缩算法概述

源：[https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html](https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html)

这里我们展示一系列压缩算法：经典Leveled，Tiered，Tiered+Leveled，Leveled-N,FIFO。除了这些，RocksDB实现了Tiered+Leveled和termed Level，Tiered termed Universal，FIFO。

## 经典Leveled

经典Leveled压缩算法，第一次在O'Neil et al的LSM-tree论文中出现，将读操作的空间放大以及写放大最小化。

LSM树是一系列的层。每一层都是一个排序结果，可以被按照范围切分成许多分片放到独立的文件中。每一层都比上一层大非常多倍。相邻层的大小倍数叫做扇出，当所有层之间的扇出都相同的时候，写放大会被最小化。把数据压缩进第N层（Ln）会把地N-1层（Ln-1）的数据合并到Ln。压缩到Ln会把之前和并进Ln的数据重写。最坏情况下，每层的写放大等于扇出数，但实际操作中，通常他会比扇出数小，Hyeontaek Lim et al的论文有相关解释。

原始LSM论文中的压缩算法使用 all-to-all的——所有Ln-1的数据会跟所有Ln层的数据合并。LevelDB和RocksDB的则是some-to-some的——Ln-1的部分数据和并进Ln层的部分数据（有覆盖的部分）

尽管leveled的写放大通常比tiered要大，但是在某些场景，leveled是有优势的。首先是按key顺序插入，一个RocksDB的优化大大减少这种场景的写放大。另一个是有倾向性的写操作，导致只有一小块的key会被更新。把RocksDB的压缩优先级设置为正确的树值，压缩过程应该在层数最小的，拥有足够空间存储写操作的层停止——他不会一致写到最大层。当leveled压缩是some-to-some模式，那么压缩只会对LSM树的一个写操作覆盖了的分片进行处理，这样可以让写放大比all-to-all模式小很多。

## Leveled-N

Leveled-N跟Leveled压缩算法很像，但是会有更小的写放大，更多的读放大。它允许每层拥有大于一个排序结果。压缩合并所有Ln-1的排序结果到Ln的一个排序结果中，也就是Leveled。然后"-N“会被驾到名称中用于暗示每层可能会有n个排序结果。Dostoevsky的论文定义了一个压缩算法，名称是Fluid LSM，他最大的层有一个排序结果，但是非最大层有多于一个排序结果。leveled压缩会在最大层完成

## Tiered

Tiered压缩通过增加读放大和空间放大，来最小化写放大。



