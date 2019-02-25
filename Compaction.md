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

LSM树仍旧可以看成是Niv Dayan和Stratos Idreos论文中讲到的一系列的层。么一层有N个排序结果。每个Ln层的排序结果都比上一层的排序结果大n倍。压缩合并同一层的所有排序结果来构造一个下一层的新的排序结果。这里的N与leveled压缩的扇出类似。合并到下一层的时候，压缩不会读/重写已经排序好的Ln层的结果。每层的写放大是1，远小于Leveled的扇出。

一个比较接近Tiered的实现是合并相似大小的排序结果，不必关心层的概念（这个概念会引入一个特定大小的排序结果的目标数字）。大多数（实现）包含一些主压缩的概念，也就是包含最大的排序结果，然后还有一些条件用来触发主，和非主压缩。通常的情况是会导致大量的文件以及自己。

tiered压缩也有一些挑战：

- 当压缩包含一个最大层的排序结果的时候，会有一个短暂的空间放大。
- 对于那些比较大的排序结果，排序结果的块索引和bloom过滤器会比较大。把他们切小块点通常是个好主意。
- 大的排序结果进一步压缩会花费非常多的时间。多线程可能有帮助
- 压缩过程是all-to-all的。如果写入有倾向性，并且大多数key都不更新，那么大量的排序结果都可能因为all-to-all的压缩过程倍重写。在传统的tiered算法中，没办法只重写一个大排序结果的一个子集。

对于tiered压缩，层的概念通常是一个用于构成LSM树形状的概念，以及用于估算写放大。对于RocksDB而言，他们还是一个开发细节。L0之上的层在LSM树中可以用来存储更大的排序结果。这样做的好处是可以吧排序结果切分成更小的SST文件。这减少了最大的bloom过滤器以及块索引块的大小——这对于块索引更友好——并且在分片索引/过滤倍支持之前是一个非常重要的主意。如果引入子压缩，就可以使对大块排序结果进行多线程压缩变为可能。注意RocksDB使用“universal”而不是tiered这个名字。

Tiered压缩算法在RocksDB的代码里倍命名为“Universal压缩”。

## Tiered+Leveled

Tiered+Leveled会有比leveled更小的写放大，以及比teired更小的空间放大。

Tiered+Leveled实现方式是一种混合实现，在小的层使用tiered，在大的层使用leveld。具体哪一层切换tiered和leveled可以非常灵活。现在我假定如果Ln是leveled那么所有之后的层（Ln+1,Ln+2）都是leveled。

VLDB2018的SlimDB是一个tiered+leveled的例子，机关它允许Lk层使用tiered，Ln使用leveled，而k>n。Fluid LSM倍描述为tiered+leveled的实现，但是我认为它是Leveled-N。

RocksDB中的Leveled压缩也是Tiered+Leveled。遵照max_write_buffer_number的设置，可能会有N个排序结果在memtable这一层——只有一个是活跃可写的，剩下的都是只读，等待落盘的。一个memtable落盘过程类似于tiered压缩——memtable的输出在L0构建一个新的排序结果并且不需要读/重写L0上已经存在的排序结果。根据level0_file_num_compaction_trigger的配置，L0可以有多个排序结果。所以L0是Teired的。memtable层没有压缩，所以也就没有该层是tiered还是leveled的说法。RocksDB中L0的子压缩过程会更加有趣，但这是另一篇文章的内容了。

## FIFO

FIFO 风格的压缩在淘汰的时候把最老的文件丢弃，可以被用于缓存数据。

## 选项

这里我们给出选项的概述以及他们如何影响压缩：

- Options::compaction_style —— RocksDB目前支持两种压缩算法——Universal风格和Level风格。这个选项在这两个之间切换。可以是kCompactionStyleUniversal或者kCompactionStyleLevel。如果是kCompactionStyleUniversal，你可以用Options::compaction_options_universal配置universal风格参数。
- Options::disable_auto_compactions——关闭自动压缩。你仍然可以选择手动压缩。
- Options::compaction_filter——允许应用在后台压缩的时候修改/删除一个键值对。如果希望针对不同的压缩过程，使用不同的过滤器，客户需要提供一个compaction_filter_factory。用户只能声明一种压缩过滤器或者工厂。
- Options::compaction_filter_factory——一个用于提供允许应用在后台压缩的时候修改/删除一个键值对的过滤器的工厂。

其他会影响压缩性能以及触发条件的选项是：
- Options::access_hint_on_compaction_start——压缩启动的时候，声明文件访问模式。对于该压缩的所有文件都会应用这个选项。默认：NORMAL
- Options::level0_file_num_compaction_trigger——触发level0压缩发生的文件数量。一个负数表示level-0压缩不会因为文件数量而被触发。
- Options::target_file_size_base与Options::target_file_size_multiplier——压缩的目标文件大小。target_file_size_base是level-1的每个文件的大小。Level-L目标文件的大小可以通过`target_file_size_base * (target_file_size_multiplier ^ (L-1)) `来计算。比如，如果target_file_size_base为2MB，target_file_size_multiplier为10，那么level-1的每个文件大小就为2MB，level2每个文件的大小就是20MB，level3的文件就是200MB。默认的target_file_size_base为64MB，target_file_size_multiplier为1。
- Options::max_compaction_bytes——所有压缩后的文件的最大大小。如果需要压缩的文件总大小大于这个值，我们在压缩的时候会避免展开更低级别的文件。
- Options::max_background_compactions——后台并发执行的最大线程数，会提交给默认优先级为LOW的线程池。
- Options::compaction_readahead_size——如果非零，我们在压缩的时候会做更大的读。如果你在机械硬盘上运行RocksDB，你应该把这个值设置为至少2MB。如果你不使用直接IO，我们会强制设置这个为2MB。

压缩还可以人工触发，参考 [人工触发压缩](Manual-Compaction.md)

参考rocksdb/options.h了解更多的选项信息。

## Leveled风格压缩

参考[Leveled-Compaction](Leveled-Compaction.md)

## Universal风格压缩

关于Universal风格的压缩的描述，参考[Universal-Compaction-Style](Universal-Compaction.md)

如果你正在使用Universal风格的压缩，有一个CompactionOptionsUniversal对象，会持有该风格的所有特殊压缩配置。额外的定义在rocksdb/universal_compaction.h，你可以在Options::compaction_options_universal中设置他。我们在这里简单介绍Options::compaction_options_universal：

- CompactionOptionsUniversal::size_ratio —— 比较文件大小的时候的灵活性比例。如果候选文件的大小比下一个文件小1%，那么把下一个文件也包括进压缩候选集。默认：1
- CompactionOptionsUniversal::min_merge_width —— 一次压缩中最小的文件数量。默认：2
- CompactionOptionsUniversal::max_merge_width —— 一次压缩中最大的文件数量。默认：UINT_MAX
- CompactionOptionsUniversal::max_size_amplification_percent —— 空间放大被定义为一个byte的数据存储在硬盘上需要多少额外的存储空间。例如，一个空间放大为2%意味着一个持有100byte用户数据的数据库，需要占用102byte的物理存储。通过这个定义，一个完全压缩的数据库的空间放大为0%。RocksDB用下面公式计算空间放大率：假设所有文件，除了最早的文件以外，都计算进空间放大中。默认200，这意味着一个100byte的数据库可能需要占用300byte存储。
- CompactionOptionsUniversal::compression_size_percent —— 如果这个选项被设置为-1（默认值），所有的输出文件都会根据指定的压缩（compress）类型进行压缩（compress）。如果这个选项不是负数，我们会尝试确保压缩（compress）大小刚好比这个值大。正常情况下，至少这个比例的数据会被压缩。当我们把压缩（compact）到一个新的文件，他是不是需要被压缩（compress）的标准是这样的：假设根据生成时间排序的文件列表如下:[A1....An B1....Bm C1....Ct]，A1是最新的，而Ct时最老的，我们将把C1...Ct压缩[compact]，我们计算所有的文件大小为总的大小total_size，我们把C1...Ct的总大小计算为total_C，如果total_C / total_size < compression_size_percent，压缩（compact）输出的文件会被压缩（compress）。
- CompactionOptionsUniversal::stop_style —— 停止选取下一个文件的算法条件。可以是kCompactionStopStyleSimilarSize（选择相似大小的文件）或者kCompactionStopStyleTotalSize（选取的文件的总大小>下一个文件）。默认为kCompactionStopStyleTotalSize

## FIFO压缩风格

参考 [FIFO压缩风格](FIFO-compaction-style.md)

## 线程池

压缩过程在线程池中进行，参考[线程池]()



