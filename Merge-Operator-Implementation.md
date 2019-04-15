这个页面描述了RocksDB的Merge功能的内部实现细节。这篇文章面向高级Rocksdb工程师以及/或其他Facebook中对Merge如何工作感兴趣的工程师。

**如果你是一个RocksDB的用户，并且你只是希望知道如何在生产环境使用Merge，参考[客户端接口](#Merge-Operator.md)页面。否则，这里假设你已经度过这个页面**

# 内容

下面是我们为了实现Merge进行的代码修改的高度概括：

- 我们创建了一个抽象类，名为MergeOperator，用户需要继承这个基类。
- 我们更新了Get，iteration以及Compaction()调用路径，以便在必要的时候调用MergeOperator的FullMerge()和PartialMerge()函数。
- 主要需要做的修改是实现"入栈的"合并运算元，我们会在下面描述。
- 我们引入了一些其他的接口变更（比如，更新了Options类以及DB类以支持MergeOperator）
- 我们创建了一个更简单的AssociativeMergeOperator以帮助用户简化一些通用使用场景。注意这个可能降低效率

对于读者，如果上面的描述不足以在一个高层次进行描述，你可能应该先阅读[客户端接口](#Merge-Operator.md)。否则，我们直接进入下面的细节，并聊一些设计决定和依据，以解释我们最终的实现。

# 接口

这里快速过一下接口（假设读者已经熟悉这部分内容）：

```cpp
// The Merge Operator
//
// Essentially, a MergeOperator specifies the SEMANTICS of a merge, which only
// client knows. It could be numeric addition, list append, string
// concatenation, edit data structure, ... , anything.
// The library, on the other hand, is concerned with the exercise of this
// interface, at the right time (during get, iteration, compaction...)
class MergeOperator {
 public:
  virtual ~MergeOperator() {}

  // Gives the client a way to express the read -> modify -> write semantics
  // key:         (IN) The key that's associated with this merge operation.
  // existing:    (IN) null indicates that the key does not exist before this op
  // operand_list:(IN) the sequence of merge operations to apply, front() first.
  // new_value:  (OUT) Client is responsible for filling the merge result here
  // logger:      (IN) Client could use this to log errors during merge.
  //
  // Return true on success, false on failure/corruption/etc.
  virtual bool FullMerge(const Slice& key,
                         const Slice* existing_value,
                         const std::deque<std::string>& operand_list,
                         std::string* new_value,
                         Logger* logger) const = 0;

  // This function performs merge(left_op, right_op)
  // when both the operands are themselves merge operation types.
  // Save the result in *new_value and return true. If it is impossible
  // or infeasible to combine the two operations, return false instead.
  virtual bool PartialMerge(const Slice& key,
                            const Slice& left_operand,
                            const Slice& right_operand,
                            std::string* new_value,
                            Logger* logger) const = 0;

  // The name of the MergeOperator. Used to check for MergeOperator
  // mismatches (i.e., a DB created with one MergeOperator is
  // accessed using a different MergeOperator)
  virtual const char* Name() const = 0;
};

```

# RocksDB数据模型

在详细解释merge如何工作之前，我们先简单介绍一下RocksDB的数据模型。

简而言之，RocksDB是一个带版本号的kv存储。每个对DB的修改都会有一个全局排序并且单调递增的序列号。对于每个key，RocksDB保留操作的历史。我们用OPi表示每个操作。一个key(K)经历了n次修改，逻辑上大概这样（物理上，这些修改可能会在活跃memtable，不可变memtable或者level文件）：

K:   OP1   OP2   OP3   ...   OPn

一个操作有三个属性：他的类型——要么是一个Delete，要么是一个Put（现在我们还有Merge），他的序列号，和他的值（Delete可以被认为是一个没有值的退化场景）。序列号会不断增加，但是对单一的key不保证连续性，因为他们在全局被所有的key共享。

当一个客户端发起 db->Put 或者 db->Delete 请求，这个库简单地把这个操作追加到历史记录里。不会检查这个key是否存在，大概是为了性能考虑吧（怪不得Delete在key不存在的时候也不会报错了。。。）

那么db->Get呢？他返回某个时间点的key的值，时间点通过序列号指定。key的状态可以是不存在，或者是一个无法理解的字符数值。他从不存在开始。每个操作吧key移动到一个新的状态。在这个场景，每个key是一个使用操作进行状态转移的状态机。

从状态机的视角看，Merge是一个通用状态转移操作，他先确认当前状态（已有值，或者不存在），然后与运算元（Merge操作引入的值）结合，然后生成一个新值（状态）。Put，是Merge的一个退化场景。Delete则更进一步——他甚至没有运算元，并且总是把key带回到他的原始状态——不存在。

# Get

实践中，Get返回一个key在特定时间的状态。

```
K:   OP1    OP2   OP3   ....   OPk  .... OPn
                            ^
                            |
                         Get.seq
```

假设OPk是Get可以看到的最后一次操作：

k = max(i) {seq(OPi) <= Get.seq}

那么如果OPk是一个Put或者Delete，Get只需要简单返回(Put指定的)值或者(因为Delete导致)NotFound状态。他可以忽略前面的值。

对于新的Merge操作，我们需要向后看。我们要看多远呢？直到一个Put或者Delete（之后的历史无所谓了）。

```
K:   OP1    OP2   OP3   ....    OPk  .... OPn
            Put  Merge  Merge  Merge
                                 ^
                                 |
                              Get.seq
             -------------------->
```

以上面的情况为例，Get应该返回这样的东西：

Merge(...Merge(Merge(operand(OP2), operand(OP3)), operand(OP4)..., operand(OPk))))

在内部，RocksDB会根据key的历史，从新到旧进行处理。RocksDB内部数据结构支持一个很好的“二分搜索”风格Seek函数。所以，提供一个序列号，他可以高效地返回：k = max(i) {seq(OPi) <= Get.seq}。然后从OPk开始，他会遍历历史，直到发现一个Put或者Delete。

为了真正地做好合并，rocksdb使用两个特殊的MergeOperator方法：FullMerge() 和 PartialMerge()。客户接口页提供了一个很好的概述，展示这些函数在上层的意义。但是，为了保证完整性，需要知道PartialMerge是一个可选函数，用于合并两个merge操作（运算元）为一个运算元。例如，合并OP(k-1)和OPk以生成OP'，他也是一个合并操作类型。一旦PartialMerge不能合并两个运算元，他返回false，告诉rocksdb让他自行处理运算元。怎么做的？好吧，内部，rocksdb提供一个内存栈型数据结构（我们实际使用了STL的Deque）来堆叠操作元，维护他们的相对顺序，直到一个Put/Delete操作，此时FullMerge被用于在基础值之上处理运算元列表。

Get的算法如下：

```Get(key):
  Let stack = [ ];       // in reality, this should be a "deque", but stack is simpler to conceptualize for this pseudocode
  for each entry OPi from newest to oldest:
    if OPi.type is "merge_operand":
      push OPi to stack
        while (stack has at least 2 elements and (stack.top() and stack.second_from_top() can be partial-merged)
          OP_left = stack.pop()
          OP_right = stack.pop()
          result_OP = client_merge_operator.PartialMerge(OP_left, OP_right)
          push result_OP to stack
    else if OPi.type is "put":
      return client_merge_operator.FullMerge(v, stack);
    else if v.type is "delete":
      return client_merge_operator.FullMerge(nullptr, stack);
      
  // We've reached the end (OP0) and we have no Put/Delete, just interpret it as empty (like Delete would)
  return client_merge_operator.FullMerge(nullptr, stack);

```

因此，Rocksdb会堆叠操作直到他遇到一个Put或者一个Delete（或者key历史的开始处），然后会按照顺序把数据当参数，调用用户定义的FullMerge操作。在上面的例子，他从OPk开始，然后OPk-1，。。。，等等。当Rocksdb遇到OP2，他会有一个像[OP3, OP4, ..., OPk]的合并运算元栈（OP3在栈顶）。他之后会调用用户定义的MergeOperator::FullMerge(key, existing_value = OP2, operands = [OP3, OP4, ..., OPk])。这个结果会返回给用户。

# 压缩

这一部分比较有趣，rocksdb的最重要的后台线程。压缩是一个减少key历史数据，但是不影响外部观察结果的过程。什么事外部观察结果？基本上， 通过一个指定序列号的一个快照。举个例子：

```
K:   OP1     OP2     OP3     OP4     OP5  ... OPn
              ^               ^                ^
              |               |                |
           snapshot1       snapshot2       snapshot3
```

对于每个快照，我们可以定义Supporting操作为最后可以被快照观察到的的操作。（OP2是snapshot1的Supporting操作，OP4是snapshot2的Supporting操作。。。）

显然，我们没法删除任何Supporting操作，而不影响外部观测结果。那么其他操作呢？在引入Merge前，我们可以丢弃所有非Supporting操作。在上面的例子，一个全压缩可以删除K的历史，直到只剩OP2 OP4和OPn。原因很简单，Put和Delete操作都是快捷操作，他们隐藏之前的操作。

而merge，过程就不一样了。尽管有些运算元不是任何快照的Supporting操作，我们还是不能简单删除它，因为后续的merge操作可能依赖他来生成结果。同事，实际上，这意味着我们甚至不能丢弃之前的Put或者Delete操作，因为他们可能被后续的merge依赖。

那么我们怎么办呢、我们从最新的处理到最老的，“堆叠”（以及/或者PartialMerging）merge运算元。在下面的任意条件成立的时候（看谁先发生），我们停止堆叠过程然后处理栈

- 一个Put/Delete发生——我们调用FullMerge(值或者nullptr,stack)
- key的历史结束——我们调用FullMerge(nullptr, stack)
- 遇到一个Supporting操作（快照）——参考下面
- 文件结束——参考下面

签名两个例子有点类似于Get()。如果你看到一个Put，调用FullMerge(put的值, stack)。如果你看到一个delete，类似。

压缩引入两个新的场景。首先，如果遇到一个快照，我们必须停止合并过程。当这个发生，我们简单写出未合并的运算元，清空栈，然后继续压缩（从Supporting操作开始）。类似的，如果我们完成了压缩（“文件结束”），我们不能简单执行FullMerge(nullptr, stack)，因为我们可能没有看到key的历史的开始；可能有一些文件刚好没有被包含到这一次压缩中来。因此，在这种场景，我们也只能简单写出未合并的运算元，然后清空栈。两种例子中，所有合并运算元都变成了类似于“Supporting操作”的东西，不能被丢弃。

这里 PartialMerge 的角色则是为了加快压缩。由于他可以支持类似于“文件结束”的场景，可能大多数的合并运算元都可以被亚索吊。因此，支持 Partial Merge的Merge Operator对压缩来说更简单，因为剩下的运算元不会被堆叠，而是在写出到新文件前合并为一个运算元。

# 例子

我们来介绍一个实际的例子，以解释上面的规则。比如有一个计数器K，从0开始，经过一系列的Add操作，被重置为2，然后继续更多的Add操作。现在一个全量压缩发生（同时还有一些外部可见的快照）—— 会发生什么呢？

```
K:    0    +1    +2    +3    +4     +5      2     +1     +2
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3
```
我们一步步看, 我们从最新的操作扫描到最老的操作
```

K:    0    +1    +2    +3    +4     +5      2    (+1     +2)
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3

```

一个Merge操作消费了之前的Merge操作，生成了一个新的Merge操作（或者一个栈）(+1  +2) => PartialMerge(1,2) => +3

```
      

K:    0    +1    +2    +3    +4     +5      2            +3
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3

K:    0    +1    +2    +3    +4     +5     (2            +3)
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3
```
一个合并操作消费了之前的Put操作，然后生成了一个新的Put操作
      (2   +3) =>  FullMerge(2, 3) => 5
```
K:    0    +1    +2    +3    +4     +5                    5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3
```
一个新处理的Put操作结果仍旧是Put，因此隐藏所有非Supporting操作
      (+5   5) => 5
```

K:    0    +1    +2   (+3    +4)                          5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3

(+3  +4) => PartialMerge(3,4) => +7

K:    0    +1    +2          +7                           5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3
```
Merge操作无法消费之前的Supporting操作  (+2   +7) 无法合并
```
K:    0   (+1    +2)         +7                           5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3

(+1  +2) => PartialMerge(1,2) => +3

K:    0          +3          +7                           5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3

K:   (0          +3)         +7                           5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3

(0   +3) => FullMerge(0,3) => 3

K:               3           +7                           5
                 ^           ^                            ^
                 |           |                            |
              snapshot1   snapshot2                   snapshot3
```


总结起来：压缩中，如果一个Supporting操作是Merge，他会（通过PartialMerge或者堆叠）和之前的操作组合，直到：

- 遇到另一个PartialMerge操作（换句话说，我们跨过了快照的边界）
- 遇到一个Put或者一个Delete操作，我们把Merge操作转换为Put操作
- 遇到key历史的结束，我们把Merge操作转换为Put
- 遇到压缩文件的结束，我们认为跨越了快照边界

注意上面的例子，我们假设merge操作定义了PartialMerge。对于没有定义PartialMerge的操作，运算元会被组合刀一个栈中，直到其中一个条件触发。

# 这个压缩模型的问题

如果Put/Delete没有找到，比如，如果Put/Delete发生在另一个文件，没有被压缩，那么压缩会简单地一个个地写出key，就好像他们没有被压缩一样。主要的问题是我们做了很多无用功来把它们压入deque。

类似的，如果有一个单一的key，有 大量 merge 操作应用于他，如果没有PartialMerge，那么所有这些操作必须被存储在内存。最后，可能会导致内存溢出或者类似的问题。

**未来的可能方案**：为了避免为何stack/deque的内存使用，可能遍历这个列表两次会更好，一次向前找到一个Put/Delete，然后一次反向。这可能会需要大量的磁盘IO，但是这只是一个建议。最后我们决定不这么做，因为大多数（如果不是全部）工作符合下，内存处理应该对每个单独的key都是足够的。未来，可以围绕这个进行讨论和压力测试。

# 压缩算法

算法上，压缩现在这么工作：

```
Compaction(snaps, files):
  // <snaps> is the set of snapshots (i.e.: a list of sequence numbers)
  // <files> is the set of files undergoing compaction
  Let input = a file composed of the union of all files
  Let output = a file to store the resulting entries

  Let stack = [];       // in reality, this should be a "deque", but stack is simpler to conceptualize in this pseudo-code
  for each v from newest to oldest in input:
    clear_stack = false
    if v.sequence_number is in snaps:
      clear_stack = true
    else if stack not empty && v.key != stack.top.key:
      clear_stack = true

    if clear_stack:
      write out all operands on stack to output (in the same order as encountered)
      clear(stack)

    if v.type is "merge_operand":
      push v to stack
        while (stack has at least 2 elements and (stack.top and stack.second_from_top can be partial-merged)):
          v1 = stack.pop();
          v2 = stack.pop();
          result_v = client_merge_operator.PartialMerge(v1,v2)
          push result_v to stack
    if v.type is "put":
      write client_merge_operator.FullMerge(v, stack) to output
      clear stack
    if v.type is "delete":
      write client_merge_operator.FullMerge(nullptr, stack) to output
      clear stack

  If stack not empty:
    if end-of-key-history for key on stack:
      write client_merge_operator.FullMerge(nullptr, stack) to output
      clear(stack)
    else
      write out all operands on stack to output
      clear(stack)

  return output
```

# 压缩过程中选择上层文件

注意，所有的合并运算元的相对顺序应该固定。由于所有迭代器”一层层地“搜索数据库，我们不希望在早期的level发现新旧运算元顺序相反的情况。所以，我们需要更新压缩过程，当他选择压缩的文件的时候，他拓展他的上层文件，以包含所有”更早的“合并运算元。为什么改变这个？因为当任何项目被压缩，他总是移动到一个更低的层。所以如果对于给定的key，其合并运算元分散在同一层的各个文件，但是只有一些进入了压缩，那么有可能出现一种奇怪的情况，就是更新的key被压缩到了更底层的level。

** 技术上来说，这是经典rocksdb的一个bug！ ** 特别地，这个问题总是在那，但是对于大多数不使用合并运算的应用，可以假设每个层都有一个key的一个版本（除了level0），因为压缩总是合并重复的Put，只要简单的记录最后的put 的值。因此这个交换顺序的概念无关紧要，除了在level-0（压缩总是包括所有有交集的文件）。所以这被修复了。

**一些问题：**在恶意输入下，这会导致总是要在压缩的时候包含大量文件，即使系统只希望挑选一个文件。这可能会降低速度，但是压力测试显示这不会是个问题。

# 效率相关的笔记

关于merge和部分merge的性能的快速讨论。

**使用一个运算元的栈可以更高效**。比如，一个追加字符串的例子（假设没有部分merge），给用户提供一个堆叠的string运算元集合来追加允许用户分散构造最终字符串的压力。例如，如果我给出了一个1000个需要追加的小字符串的列表，我可以使用这些字符串来计算最后的字符串的大小，预分配空间，然后处理并拷贝所有数据到新分配的内存数组。作为比较，如果我强制要求总是部分合并，系统需要执行1000次重新分配，开销非常大。在多数使用场景，这可能不是问题；在我们的压力测试，我们发现这只对一个巨大的有热点key的内存数据库有影响，但是这需要考虑”增长数据“。在所有场景，我们都给用户提供了选择。

最重要的是，存在使用运算元栈（而不是一个运算元）提供一个显著增加操作运算效率的方法。比如，在上面的追加字符串的例子，合并运算符可以让字符追加操作做到大概O(N)时间（N是最后字符串的长度），而不使用栈，我们需要使用O(N^2)时间复杂度。

更多信息，请联系rocksdb团队，或者参考RocksDB的wiki页。

