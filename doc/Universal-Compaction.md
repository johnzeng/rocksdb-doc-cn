Universal压缩方式是一种压缩方式，面向那些需要用读放大和空间放大，来换取更低的写放大的场景。

# 概念基础

就如很多系统和作者介绍的，这里有两个类型的LSM树压缩策略：

- leveled压缩，这个是RocksDB的默认策略
- 一种替代压缩策略，有时候被叫做“size tiered”[1] 或者“tiered”[2]

两种策略的主要差别在于，leveled压缩倾向于更加频繁地把小的排序结果合并到大的里面，而“tiered”等待多个大小接近的排序结果，然后把它们合并到一起。

通常认为第二种策略提供更好的写放大表现，但是读放大会变糟糕[2][3]。凭直觉来看：在tiered存储，每当一个更新被压缩，他倾向于从小的排序结果移动到一个大很多的排序结果。每次压缩都可能会指数级接近最后那个，最大的排序结果。然而在leveled压缩，一次更新更加可能通过一个小的排序结果，被合并进一个更大的排序结果，而不是直接作为一个较小的排序结果的一部分保存起来。就结果而言，大多数时候一个更新被压缩，而不是被移动到一个更大的排序结果，他没有往最大的排序结果前进多少。

“tiered”压缩不是没有坏处的。最坏的情况下的排序结果数量会比leveled多很多。这会导致读数据的时候有更高的IO和/或者更高的CPU消耗。压缩调度天然的懒调度模式，同样会导致压缩的时候更多的毛刺，排序结果的数量会差异非常大，所以性能很不稳定。

尽管如此，RocksDB仍旧提供了“tiered”家族的算法，Universal压缩。用户在leveled压缩无法处理想要的写速率的时候，可以尝试这种压缩方式。

[1] 这个被Cassandra使用，参考他们的[文档](https://docs.datastax.com/en/cassandra/3.0/cassandra/operations/opsConfigureCompaction.html)
[2] N. Dayan, M. Athanassoulis 与 S. Idreos, [“Monkey: Optimal Navigable Key-Value Store,”](https://stratos.seas.harvard.edu/publications/monkey-optimal-navigable-key-value-store) 在 ACM SIGMOD International Conference on Management of Data, 2017发布.
[3] [https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html](https://smalldatum.blogspot.com/2018/08/name-that-compaction-algorithm.html)


# 概述以及基本思想

使用这个压缩风格的时候，所有的SST文件都以排序结果覆盖所有key范围的方式组织。一个排序结果覆盖在一段时间范围内生成的数据。不同的排序结果的时间范围不会重叠。压缩只能发生在两个或者更多的连贯时间的排序结果。输出是一个单一排序结果，他的时间范围是所有输入排序结果的组合。任何一个压缩后，排序结果不会相互覆盖时间的假定仍旧成立。一个排序结果可以实现为一个L0的文件，或者一个key按照文件进行分片排序的“层”

这种压缩风格的基本思想为：排序结果的数量阈值N，我们只在排序结果数量达到N的时候开始压缩。当这个发生的时候，我们尝试挑选文件进行压缩，这样排序结果数量会以最经济环保的方式减小：1，从最小的文件开始；2，如果下一个文件的大小不大于已经有的压缩大小，把这个文件包含进来。这种策略假设，并且尝试维持一个 排序结果包含更加新的数据会比拥有旧的数据的排序结果小 这种特性。

# 限制

## 双倍大小问题

在universal风格压缩中，有时候全压缩是需要的。这种情况下，输出数据大小接近于输入大小。在压缩过程中，输入文件和输出文件都需要保留，这样DB会暂时的使用两倍磁盘空间。请确保你有足够的空间进行全量压缩。

## num_levels=1时DB(列族)的大小

当使用universal压缩，如果num_levels=1，DB所有的数据（或者更准确来说，列族）有时候会被压缩进一个SST文件中。一个SST文件的大小是有限制的。在RocksDB里，一个块不能超过4GB（这样size才能使uint32）。如果一个SST文件过大，索引块可以超过这个大小。索引块的大小取决于你的数据。我们其中的一个使用场景中，我们观察到如果DB数据增长到大概250GB，使用4K数据块大小，DB会到达这个限制。可以使用[分片索引]()解决这个问题

如果用户设置num_levels比1大，也可以解决问题。在这是，更大的“文件”会被放到更大的“层”，里面文件切分成更小的文件（细节在下面描述）。 L0->L1压缩还是会在并行压缩发生，但是L0的文件可能会更小。

# 数据分布和排列

## 排序结果

就像上面说的一样，数据以排序结果的形式组织。排序结果根据里面的数据的更新时间排布和排序在L0的文件里或者作为一个完整的“层”。

这里有一个典型的文件排布例子：

```
Level 0: File0_0, File0_1, File0_2
Level 1: (empty)
Level 2: (empty)
Level 3: (empty)
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

更大数字的层有更老的排序结果。在这个例子里，有5个排序结果：level0,level4和level5。Level5是最老的排序结果，level4更新一些，level0的文件最新。

## 压缩输出的位置

压缩调度的时候总是选连续时间范围内的排序结果，并且输出的总是另一个排序结果。根据更老的数据在更大的层的规则，我们总是把压缩输出放在尽可能高的层。

使用上面的例子，我们有下面的排序结果：

```
File0_0
File0_1
File0_2
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

如果我们压缩所有文件，输出的排序结果会被放在level5，就变成了：

```
Level 5: File5_0', File5_1', File5_2', File5_3', File5_4', File5_5', File5_6', File5_7'
```

从这个状态开始，如果我们调度了其他压缩，看看怎么放压缩结果：

如果我们压缩File0_1, File0_2 和 Level 4，输出的排序结果会被放在level4.

```
Level 0: File0_0
Level 1: (empty)
Level 2: (empty)
Level 3: (empty)
Level 4: File4_0', File4_1', File4_2', File4_3'
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

如果我们压缩File0_0, File0_1 和 File0_2，输出结果会被放在level3

```
Level 0: (empty)
Level 1: (empty)
Level 2: (empty)
Level 3: File3_0, File3_1, File3_2
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

如果我们压缩File0_0 和 File0_1，输出的排序结果仍旧会被放在level0

```
Level 0: File0_0', File0_2
Level 1: (empty)
Level 2: (empty)
Level 3: (empty)
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```


## 特别情况options.num_levels=1

如果options.num_levels=1，我们仍旧遵照同样的摆放规则。这意味着所有文件会被放在level0下，每个文件是一个排序结果。行为会与原始universal排序相同，所以他可以用作向后兼容模式。

# 压缩挑选算法

假设我们有排序结果

```
R1, R2, R3, ..., Rn
```

R1有DB最新的更新，而Rn有最老的DB更新。注意排序结果内是按照数据的年龄排序的，不是排序结果本身。根据这个排序结果，压缩之后，输出的排序结果总是放在输入的位置。

压缩是怎么选择文件的：

## 前置条件：n >= options.level0_file_num_compaction_trigger

除非排序结果到达这个阈值，否则不会触发压缩。

（注意，尽管这个选项名使用的是“file”，因为历史原因，触发条件实际统计的是排序结果。对于下面所有的选项，因为同样的原因，“file”同样意味着排序结果）

如果前置条件满足了，还有三个条件。他们分别触发一个压缩：

## 1. 有空间放大触发的压缩

如果预计的空间放大比例大于options.compaction_options_universal.max_size_amplification_percent / 100，所有的文件会被压缩到一个排序结果。

这里展示空间放大比例是如何算的：

```
size amplification ratio = (size(R1) + size(R2) + ... size(Rn-1)) / size(Rn)
```

注意，Rn的大小没有被包括进来，这意味着0是最完美的空间放大比例而100意味着DB大小会变成现有数据的两倍，200以为这三倍。

为什么我们这么计算空间放大：在一个稳定大小的DB中，删除速度和插入速度应该是接近的，这意味着任何一个排序结果，除了Rn，都应该包括相近数量的插入和删除。在把R1，R2。。。Rn-1合并到Rn之后，作用于他们的空间会互相抵消，这样输出的数据应该是Rn的大小，也就是线上数据的大小，也就是空间放大的基数。

如果options.compaction_options_universal.max_size_amplification_percent = 25，意味着我们会保持总的DB空间小于125%的总线上数据大小。我们用实际数值来举例。假设压缩只因为空间放大触发，options.level0_file_num_compaction_trigger = 1，每个memtable落盘大小总是1，压缩的大小总是等于总输入大小。两次落盘之后，我们有两个大小为1的文件，也就是 1/1 > 25%，所以我们需要做一次全量压缩：

1 1 => 2
另一个memtable落盘后，我们有
1 2 => 3
又一次触发一个全量压缩，因为 1/2 > 25%。同样的：
1 3 => 4
但是在下次落盘后，压缩不会被触发：
1 4
因为1/4 <= 25。另一个memtable落盘会触发另一个压缩：
1 1 4 => 6

因为 (1+1)/4 > 25%

然后后续会这样

```
1
1 1  =>  2
1 2  =>  3
1 3  =>  4
1 4
1 1 4  =>  6
1 6
1 1 6  =>  8
1 8
1 1 8
1 1 1 8  =>  11
1 11
1 1 11
1 1 1 11  =>  14
1 14
1 1 14
1 1 1 14
1 1 1 1 14  =>  18
```

# 因为个体大小比例触发压缩

我们计算一个大小比例触发值如下：

```
size_ratio_trigger = 100 + options.compaction_options_universal.size_ratio / 100
```

通常options.compaction_options_universal.size_ratio接近0所以大小比例触发接近1。

我们从R1开始，如果size(R2) / size(R1) <= size_ratio_trigger, 那么（R1，R2）会被允许压缩到一起。我们从这里继续决定R3是不是可以加进来。如果size(R3) / size(R1+r2) <= size_ratio_trigger，我们应该把它包含进来，得到(R1,R2,R3)。然后我们对R4做同样的事情。我们一直用所有已有的大小总和，跟下一个排序结果比较，直到size_ratio_trigger条件不满足。

这里举个例子来说明。假设options.compaction_options_universal.size_ratio = 0，所有memtable落盘大小总是1，压缩后的大小总是等于总输入大小，压缩只因为空间放大触发，并且options.level0_file_num_compaction_trigger = 5。从一个空DB开始，5次罗盘之后，我们有5个大小为1的文件，触发一次全量压缩因为 1/1 <= 1， 1/(1+1) <=1，1/(1+1+1) <=1 且 1/(1+1+1+1) <= 1：

1 1 1 1 1 => 5

又过了4个memtable落盘，凑够5个文件。前面4个文件满足合并要求1/1 <= 1, 1/(1+1) <= 1, 1/(1+1+1) <=1。但是第五个不满足5/(1+1+1+1) > 1:

1 1 1 1 5  => 4 5

他们继续这样走几个轮回：

```
1 1 1 1 1  =>  5
1 5  (不触发压缩)
1 1 5  (不触发压缩)
1 1 1 5  (不触发压缩)
1 1 1 1 5  => 4 5
1 4 5  (不触发压缩)
1 1 4 5  (不触发压缩)
1 1 1 4 5 => 3 4 5
1 3 4 5  (不触发压缩)
1 1 3 4 5 => 2 3 4 5
```

又一次落盘，形成

1 2 3 4 5

没有压缩会被触发，我们把压缩hold住。只有当另一次落盘到来，所有文件都满足压缩条件：

1 1 2 3 4 5 => 16

因为 1/1 <=1, 2/(1+1) <= 1, 3/(1+1+2) <= 1, 4/(1+1+2+3) <= 1 且 5/(1+1+2+3+4) <= 1。然后我们接着来：

```
1 16  (不触发压缩)
1 1 16  (不触发压缩)
1 1 1 16  (不触发压缩)
1 1 1 1 16  => 4 16
1 4 16  (不触发压缩)
1 1 4 16  (不触发压缩)
1 1 1 4 16 => 3 4 16
1 3 4 16  (不触发压缩)
1 1 3 4 16 => 2 3 4 16
1 2 3 4 16  (不触发压缩)
1 1 2 3 4 16 => 11 16
```

压缩只在输入的排序数量满足至少options.compaction_options_universal.min_merge_width，且输入的排序结果数量不会多于options.compaction_options_universal.max_merge_width的时候触发。

## 3. 由于排序结果数量触发压缩

如果每次我们尝试调度一个压缩，1和2都不触发，我们会尝试压缩R1,R2...中的任何排序结果，这样压缩之后，总的排序结果数量不会多于options.level0_file_num_compaction_trigger。如果我们需要压缩多于options.compaction_options_universal.max_merge_width的排序结果，我们让他不超过options.compaction_options_universal.max_merge_width。

“尝试调度”意思是在一个memtable落盘之后，一次压缩结束之后。有时候会有重复的请求被调度

参考[Universal压缩范例](https://rocksdb.org.cn/doc/Universal-Style-Compaction-Example.html)看排序结果输出在一个通用设定下是什么样的。

并行压缩在 options.max_background_compactions > 1 的时候发生。跟其他所有压缩方式一样，并行压缩不会再同一个排序结果中发生。

# 子压缩

自压缩在universal压缩中支持。如果一个压缩的输出层不是“层”0，我们会尝试把输入分片然后使用options.max_subcompaction定义的线程数来并行压缩他们。这帮助解决universal压缩全量压缩时间过长的问题。

# 调优选项

下面这些选项影响universal压缩：


- options.compaction_options_universal ： 上面提到的许多选项
- options.level0_file_num_compaction_trigger ： 任何压缩的触发条件。他还意味着，在所有压缩结束之后，排序结果的数量会小于options.level0_file_num_compaction_trigger+1
- options.level0_slowdown_writes_trigger：如果排序结果的数量超过这个数值，写入会被强制减慢。
- options.level0_stop_writes_trigger：如果排序结果超过这个值，写入会停止，直到压缩结束，并且排序结果变得比这个值低
- options.num_levels： 如果这个值为1，所有排序结果都会被放在level0的文件中。否则，我们会尝试尽可能地填充非零层。options.num_levels越大，我们约不会再level 0存放大文件
- options.target_file_size_base：如果options.num_levels > 1才有效。除了level 0以外的文件都会被裁减到不大于这个阈值的大小。
- options.target_file_size_multiplier：这是有效的，但是我们没有找到一个合理的场景来使用这个选项。所以我们不推荐你调这个选项

下面这些选项 **不会** 影响universal压缩：

- options.max_bytes_for_level_base: 只影响level-based压缩
-	options.level_compaction_dynamic_level_bytes: 只影响level-based压缩
-	options.max_bytes_for_level_multiplier 与 options.max_bytes_for_level_multiplier_additional: 只影响level-based压缩
-	options.expanded_compaction_factor: 只影响level-based压缩
- options.source_compaction_factor: 只影响level-based压缩
-	options.max_grandparent_overlap_factor: 只影响level-based压缩
-	options.soft_rate_limit 与 options.hard_rate_limit: 已被丢弃
-	options.hard_pending_compaction_bytes_limit: 只影响level-based压缩
-	options.compaction_pri: 只在level-based支持

# 估算写放大

调优universal压缩的时候，估算写放大将非常有帮助。然而，这很难。由于universal压缩总是做局部最优选择，LSM树的结构非常难预计。你可以参考上面提到的例子。我们还没有一个好的数学模型来预计写放大。

这里有一个不那么好的估计。

这个估算基于一个简单的假设，每一次一个更新被压缩，输出文件都是原始文件的两倍（这是一个拍脑袋做出的假设），除了第一个或者最后一个压缩，这两种情况，相似大小的压缩在一起的排序结果。

举个例子，如果options.compaction_options_universal.max_size_amplification_percent = 25，最后一个排序结果的大小为256GB，从memtable落盘的时候，SST文件的大小是256MB，options.level0_file_num_compaction_trigger = 11，然后在一个稳定的阶段，文件大小会这样：

256MB
256MB
256MB
256MB
2GB
4GB
8GB
16GB
16GB
16GB
256GB

压缩阶段，写放大就会这样：

```
256MB
256MB
256MB  (写放大 1)
256MB
--------
2GB    (写放大 1)
--------
4GB    (写放大 1)
--------
8GB    (写放大 1)
--------
16GB
16GB    (写放大 1)
16GB
--------
256GB   (写放大 4)
```

所以总共的写放大大概就会是9。

这里展示写放大如何被估算

options.compaction_options_universal.max_size_amplification_percent总是自行引入一个小于100的写放大。写放大大概这么估算：

WA1 = 100 / options.compaction_options_universal.max_size_amplification_percent

如果他不低于100，假设

WA1 = 0

假设除了最后一个排序结果的总数据大小为S。如果options.compaction_options_universal.max_size_amplification_percent < 100，假设使用

S = total_size * (options.compaction_options_universal.max_size_amplification_percent/100)

或者

S = total_size

假设memtable落盘的SST文件大小为M。我们估计一次更新可以引起的最大数量的压缩为：

p = log(2, S/M)

我们推荐options.level0_file_num_compaction_trigger > p。然后我们估计因为个体大小比率导致的写放大：

WA2 = p - log(2, options.level0_file_num_compaction_trigger - p)

然后我们就有了总的写放大估算值 WA1 + WA2


