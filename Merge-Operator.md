本页描述了Rocksdb里的 Read-modify-write原子操作，也叫"Merge"操作。这只是一个接口概述，面向那些提出：“什么时候我需要使用Merge，以及如何使用”的问题的客户端以及RocksDB使用者。

# 为什么

Rocksdb是一个高性能的嵌入式持久化key-value存储。它提供一些简单操作 Get,Put,Delete来实现一个优雅的 Loopup-table-like接口

通常，有一些通用的模式来更新一个已经存在的value。为了在rocksdb里面实现这个，客户端会需要读（Get）一个已经存在的值，修改，然后写（put）回db。来看一个常见例子。

想象我们在维护一系列的uint64的计数器。每个计数器都有一个不同的名字。我们希望实现四个高级操作： Set,Add,Get以及Remove。

首先，我们定义接口来保证语义正确。错误处理被放到一边以保证代码清晰。

```cpp
class Counters {
 public:
  // (re)set the value of a named counter
  virtual void Set(const string& key, uint64_t value);

  // remove the named counter
  virtual void Remove(const string& key);

  // retrieve the current value of the named counter, return false if not found
  virtual bool Get(const string& key, uint64_t *value);

  // increase the named counter by value.
  // if the counter does not exist,  treat it as if the counter was initialized to zero
  virtual void Add(const string& key, uint64_t value);
  };

```

然后，我们使用现有RocksDB接口实现它。伪代码如下：

```cpp
    class RocksCounters : public Counters {
     public:
      static uint64_t kDefaultCount = 0;
      RocksCounters(std::shared_ptr<DB> db);

      // mapped to a RocksDB Put
      virtual void Set(const string& key, uint64_t value) {
        string serialized = Serialize(value);
        db_->Put(put_option_, key,  serialized));
      }

      // mapped to a RocksDB Delete
      virtual void Remove(const string& key) {
        db_->Delete(delete_option_, key);
      }

      // mapped to a RocksDB Get
      virtual bool Get(const string& key, uint64_t *value) {
        string str;
        auto s = db_->Get(get_option_, key,  &str);
        if (s.ok()) {
          *value = Deserialize(str);
          return true;
        } else {
          return false;
        }
      }

      // implemented as get -> modify -> set
      virtual void Add(const string& key, uint64_t value) {
        uint64_t base;
        if (!Get(key, &base)) {
          base = kDefaultValue;
        }
        Set(key, base + value);
      }
    };

```
注意，除了Add操作，所有其他三个操作都可以直接映射到rocksdb的一个操作。这不是坏事。然而，一个概念上的单一操作，Add，却只能映射到两个rocksdb操作。这里也有一个性能上的隐患——随机Get在rocksdb里面相对较慢。

现在，加入我们需要维护一个Counter服务。根据现在的服务器核心数，我们的服务几乎肯定是多线程的。如果线程不是根据键空间进行分片，可能同一个计数器的多个Add操作，会被多个线程并行之行。好吧，如果我们还有严格的一致性需求（任何一个更新都不能被丢失），我们可能需要在Add外面加一个额外的同步，比如锁之类的。压力就变大了。

如果rocksdb直接支持Add操作呢？我们可以提出如下代码：

```cpp
    virtual void Add(const string& key, uint64_t value) {
      string serialized = Serialize(value);
      db->Add(add_option, key, serialized);
    }
```

这对计数器来说很合理。但你的RocksDB里面并不全是计数器。比如我们需要追踪一个用户去过哪些地方。我们可以在这个用户的key下面存储一个（序列化的）地址列表。往这个列表增加新的地址可能是一个通用操作。我们在这种情况下可能需要使用Append操作：db->Append(user_key, serialize(new_location))。这意味着read-modify-write操作的语义是由客户端定义的。为了保证库的通用性，我们最好能抽象出这个操作，然后允许客户端声明语义。这就带出了我们的提案：Merge。

# 是什么

**我们以一个一等操作的待遇在rocksdb里面开发了一个通用Merge操作来捕获read-modify-write语义**

这个Merge操作：

- 把read-modify-write语义概括成一个简单的抽象接口
- 允许用户避免引入额外的重复的Get调用。
- 使用后端优化来决定什么时候/如何在不改变语义的情况下合并操作运算元
- 可以在某些情况下分期偿还所有增量更新带来的压力，以提供有效的渐增式累加。

# 如何使用

在接下来的段落，那些客户端的代码将被说明。我们还提供一个简单的大纲来说明如何使用Merge。我们假定读者已经知道如何使用经典的Rocksdb（或者leveldb），包括：

- DB类（包括构造，DB::Put(), DB::Get(), 以及 DB::Delete()）
- Options类（以及指导如何在创建的基础上声明数据库配置）
- 知道所有写入的数据库key/value都是简单的字符串字节

## 接口概览

我们定义了一个新的接口/虚基类：MergeOperator。它导出了一些函数，告诉RocksDB如何使用基础值（Put/Delete）组合增量更新操作（又叫“合并运算元”）。这些函数还可以用来告诉rocksdb如何组合这些合并操作来构造一个新的合并操作（称为“部分[Partial]”或者“结合性[Associative]”合并）。

简单起见，我们暂时不关注Partial和非Partial的区别。我们已经提供了一个独立的接口，名为AssociativeMergeOperator，他描述并隐藏了了部分合并的所有细节。对于大多数简单的应用（比如我们的64Bit计数器），这足够了。

读者应该假设所有合并都是通过一个名为AssociativeMergeOperator的接口来处理的。这是这个公用接口：

```cpp
    // The Associative Merge Operator interface.
    // Client needs to provide an object implementing this interface.
    // Essentially, this class specifies the SEMANTICS of a merge, which only
    // client knows. It could be numeric addition, list append, string
    // concatenation, ... , anything.
    // The library, on the other hand, is concerned with the exercise of this
    // interface, at the right time (during get, iteration, compaction...)
    class AssociativeMergeOperator : public MergeOperator {
     public:
      virtual ~AssociativeMergeOperator() {}

      // Gives the client a way to express the read -> modify -> write semantics
      // key:           (IN) The key that's associated with this merge operation.
      // existing_value:(IN) null indicates the key does not exist before this op
      // value:         (IN) the value to update/merge the existing_value with
      // new_value:    (OUT) Client is responsible for filling the merge result here
      // logger:        (IN) Client could use this to log errors during merge.
      //
      // Return true on success. Return false failure / error / corruption.
      virtual bool Merge(const Slice& key,
                         const Slice* existing_value,
                         const Slice& value,
                         std::string* new_value,
                         Logger* logger) const = 0;

      // The name of the MergeOperator. Used to check for MergeOperator
      // mismatches (i.e., a DB created with one MergeOperator is
      // accessed using a different MergeOperator)
      virtual const char* Name() const = 0;

     private:
      ...
    };
```

**一些需要注意的地方**

- AssociativeMergeOperator是一个名为MergeOperator类的子类。后面我们将看到这个更加通用化的MergeOperator在特定场合将更加强大。而我们用的AssociativeMergeOperator，则更加简单明了。
- existing_value可能是nullptr。在Merge操作是一个key的第一个操作的时候非常有用。nullptr暗示着这个key没有‘已经存在的’值。这基本需要推迟到客户端决定如何在没有预设值的时候如何处理一个merge操作。客户端可以做任何合理的事情。比如， Counters::Add假设如果一个key不存在，那么他就是一个0值。
- 如果key的空间是分片的，并且不同的子空间指向不同的数据类型并且拥有不同的合并操作语义，那么我们把key传入，然后客户端就可以多路复用这个合并操作了。例如，客户端可能选择把一个用户的余额（一个数字）存在一个名为“BAL:uid”的键下面，然后把账号的活动信息（一个列表）存入一个名为“HIS:uid”的键下面，他们存在在同一个DB中。（这是不是一个好的实践其实非常有争议）。对于当前余额，数字加法是完美的合并操作；对于活动信息，我们需要一个list append操作。因此，通过把key传给Merge回调，我们使客户端支持能区分两种类型：

```cpp
    void Merge(...) {
       if (key start with "BAL:") {
         NumericAddition(...)
       } else if (key start with "HIS:") {
         ListAppend(...);
       }
     }

```

## 其他客户端可见的接口变化

为了在一个应用里使用Merge，客户端必须先定义一个类，继承AssociativeMergeOperator接口（或者MergeOperator接口）。这个对象类应该实现这个接口的所有方法，不管是不是需要合并，他们都最终会被rocksdb在合适的时机调用。这样，合并语义就完全实现了客户端定制了。

定义好这个类之后，用户需要一个特别的方法告诉rocksdb需要使用合并操作符实现合并。我们已经在DB类和Options类里面引入了额外的字段/方法来完成这个工作。

```cpp
    // In addition to Get(), Put(), and Delete(), the DB class now also has an additional method: Merge().
    class DB {
      ...
      // Merge the database entry for "key" with "value". Returns OK on success,
      // and a non-OK status on error. The semantics of this operation is
      // determined by the user provided merge_operator when opening DB.
      // Returns Status::NotSupported if DB does not have a merge_operator.
      virtual Status Merge(
        const WriteOptions& options,
        const Slice& key,
        const Slice& value) = 0;
      ...
    };

    Struct Options {
      ...
      // REQUIRES: The client must provide a merge operator if Merge operation
      // needs to be accessed. Calling Merge on a DB without a merge operator
      // would result in Status::NotSupported. The client must ensure that the
      // merge operator supplied here has the same name and *exactly* the same
      // semantics as the merge operator provided to previous open calls on
      // the same DB. The only exception is reserved for upgrade, where a DB
      // previously without a merge operator is introduced to Merge operation
      // for the first time. It's necessary to specify a merge operator when
      // opening the DB in this case.
      // Default: nullptr
      const std::shared_ptr<MergeOperator> merge_operator;
      ...
    };

```

**注意** Options::merge_operator字段是一个shared指针指向一个MergeOperator。就像上面说的，AssociativeMergeOperator继承自MergeOperator，所以我们可以在这里声明一个AssociativeMergeOperator。这也是下面的例子用的方法。

## 客户端代码修改

有了上面的接口修改，客户端现在可以实现一个新版的Counters类，直接使用内建的Merge操作了。

Counters v2:

```cpp
   // A 'model' merge operator with uint64 addition semantics
    class UInt64AddOperator : public AssociativeMergeOperator {
     public:
      virtual bool Merge(
        const Slice& key,
        const Slice* existing_value,
        const Slice& value,
        std::string* new_value,
        Logger* logger) const override {

        // assuming 0 if no existing value
        uint64_t existing = 0;
        if (existing_value) {
          if (!Deserialize(*existing_value, &existing)) {
            // if existing_value is corrupted, treat it as 0
            Log(logger, "existing value corruption");
            existing = 0;
          }
        }

        uint64_t oper;
        if (!Deserialize(value, &oper)) {
          // if operand is corrupted, treat it as 0
          Log(logger, "operand value corruption");
          oper = 0;
        }

        auto new = existing + oper;
        *new_value = Serialize(new);
        return true;        // always return true for this, since we treat all errors as "zero".
      }

      virtual const char* Name() const override {
        return "UInt64AddOperator";
       }
    };

    // Implement 'add' directly with the new Merge operation
    class MergeBasedCounters : public RocksCounters {
     public:
      MergeBasedCounters(std::shared_ptr<DB> db);

      // mapped to a leveldb Merge operation
      virtual void Add(const string& key, uint64_t value) override {
        string serialized = Serialize(value);
        db_->Merge(merge_option_, key, serialized);
      }
    };

    // How to use it
    DB* dbp;
    Options options;
    options.merge_operator.reset(new UInt64AddOperator);
    DB::Open(options, "/tmp/db", &dbp);
    std::shared_ptr<DB> db(dbp);
    MergeBasedCounters counters(db);
    counters.Add("a", 1);
    ...
    uint64_t v;
    counters.Get("a", &v);

```

用户界面的修改非常小。RocksDB后端处理剩下的部分。

# Associativity VS 非 Associativity

到现在为止，我们已经了一个相对简单的例子来维护一个计数器数据库。似乎上面说的AssociativeMergeOperator已经足够处理这种类型的操作了。例如，如果你希望用“append”操作维护一个字符串集合，那么我们目前看到的已经可以很简单地处理这个需求了。

那么为什么这些都被认为是“简单的”？好吧，我们隐式地假设了这个数据的一个特征：结合性。这意味着我们假设：

- 通过Put放入RocksDB的数据和通过Merge操作的格式相同
- 用同一个用户定义的合并操作可以将多个合并组合成一个合并操作

比如，以Counters为例。Rocksdb数据库内部将每个值存储为有序的8byte的整形。所以，当客户端调用Counters::Set（对应于 DB::Put()），参数是有相同格式的。类似的，当客户调用Counters::Add（对应于一个DB::Merge()）的时候，merge操作也是一个序列化的8-byte整形。这意味着，在客户的UInt64AddOperator里，那个`*existing_value`可能指向原始的Put()，或者意味着一个合并操作元，这不重要！在所有情况中，只要 `*existing_value`  以及数值给出，UInt64AddOperator就能用同一种方式操作：他把他们加在一起，然后计算`*new_value`。最后，这个`*new_value`会回馈给后面的合并操作，根据merge调用的顺序。

但是，RocksDB的合并操作似乎可以有更强大的功能。比如，我们希望我们的数据库存储一个json字符串集合（比如PHP对象数组）。那么在数据库里，我们希望她们可以以完全格式化的json字符串被存储并查找，但是我们可能希望“更新”操作来更新一个json对象里的某个属性。所以我们可能会写这样的代码：

```cpp
    ...
    // Put/store the json string into to the database
    db_->Put(put_option_, "json_obj_key",
             "{ employees: [ {first_name: john, last_name: doe}, {first_name: adam, last_name: smith}] }");

    ...
    
    
    // Use a pre-defined "merge operator" to incrementally update the value of the json string
    db_->Merge(merge_option_, "json_obj_key", "employees[1].first_name = lucy");
    db_->Merge(merge_option_, "json_obj_key", "employees[0].last_name = dow");

```

在上面的伪代码中，我们看到，数据会在RocksDB里面被存储为一个json字符串（映射到Put操作），但是客户端需要更新一个值，一个“javascript风格”的赋值声明串会被作为合并元被传入。数据库会按照原样存储这些字符，然后希望用户的合并操作符来处理逻辑。

现在，AssociativeMergeOperator模型就无法处理这个了，仅仅因为他假设了我们上面提到的结合律。也就是说，在这个例子里，我们需要明确声明基础值（json字符串）和合并操作元（赋值声明），而且我们没有一个直观的方法来将一个合并运算元和另一个合并运算元组合。所以这个使用离子不符合我们的“结合律”合并模型。这就是通用MergeOperator接口有用的地方。

# 通用MergeOperator接口

MergeOperator接口被设计用于支持抽象并暴露部分关键方法来在RocksDB里面实现一个提供有效方案来实现“增量更新”。就像我们上面提到的json的例子，可能基本的数据类型（Put进数据库的）可能和更新他们的运算元有完全不同的格式。而且，我们会看到导出一些合并运算元可以组合成一个单独的合并运算元的事实有时候是有好处的，但是有时候又是不好的（原文很绕，大概来说就是需要提供一个方法判断几个合并运算元是不是可以合并为一个运算元）。这都要看客户端定义的语义。MergeOperator接口提供了一个相对简单的方法来从客户端获取这些信息：

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
      // Return true on success. Return false failure / error / corruption.
      virtual bool FullMerge(const Slice& key,
                             const Slice* existing_value,
                             const std::deque<std::string>& operand_list,
                             std::string* new_value,
                             Logger* logger) const = 0;

      struct MergeOperationInput { ... };
      struct MergeOperationOutput { ... };
      virtual bool FullMergeV2(const MergeOperationInput& merge_in,
                               MergeOperationOutput* merge_out) const;

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

      // Determines whether the MergeOperator can be called with just a single
      // merge operand.
      // Override and return true for allowing a single operand. FullMergeV2 and
      // PartialMerge/PartialMergeMulti should be implemented accordingly to handle
      // a single operand.
      virtual bool AllowSingleOperand() const { return false; }
    };

```

**一些注意的点**：
- MergeOperator有两个方法，FullMerge和PartialMerge。第一个方法在Put/Delete是*existing_value (或者 nullptr)时被使用。后面的方法在组合两个合并运算元的时候（如果可以）被使用。
- AssociativeMergeOperator只需要继承MergeOperator然后提供私有的默认方法实现就行了，然后简单暴露一个包裹好的函数。
- 在MergeOperator里，FullMerge函数提供一个`*existing_value`以及一个队列(std::deque)的合并运算元，而不是单个运算元。我们后面会解释。

## 这些方法如何工作的？

在上层，需要注意到，任何调用DB::Put()或者DB:Merge都不需要强制树枝马上被计算或者合并马上发生。Rocksdb会或多或少地懒惰地决定什么时候需要执行这些操作（例如，下次用户调用Get的时候，或者当系统决定调用名为“压缩”的清理流程的时候）。这意味着，当MergeOperator真正被调用，可能会有多个“入栈的”运算元需要被执行。因此，MergeOperator::FullMerge()方法提供一个`*existing_value`以及一个压栈的运算元列表。MergeOperator 应该一个接一个地执行这些运算元（或者任何客户端决定的优化方法，保证最终`*new_value`会被按要求计算成所有运算元执行的结果）

## 部分合并VS入栈

有时候，可能在系统遇到运算元就调用合并操作会更好，而不是入栈。MergeOperator::PartialMerge就是为此准备的。如果客户声明运算符可以逻辑上处理“组合”两个运算元位一个单独的运算元，对应的语义就应该提供这个方法，然后应该返回true。如果逻辑上不行，就简单的保留`*new_value`不变，然后返回false即可。

理论上，当库决定入栈然后执行操作，他先尝试对每一对运算元执行用户定义的PartialMerge。只要这个操作返回了false，他就会被插入到栈内，直到他遇到一个Put/Delete的值，他才会调用FullMerge操作，把所有运算元当参数传入。通常来说，这个最后的FullMerge应该返回true。只有当有坏格式数据的时候才应该返回false。

## AssociativeMergeOperator怎么做的？

就像上面说的，AssociativeMergeOperator继承自MergeOperator并且允许用户声明一个单独的合并操作。他覆盖了PartialMerge和FullMerge让他们使用AssociativeMergeOperator::Merge()。所以他才可以在合并运算元和设置基础值的时候使用。这也是为什么他只能在符合“结合律”的假设的时候使用。

## 什么时候允许单一合并元

基本来说，合并运算只会在有两个合并运算元的时候被调用。覆盖AllowSingleOperand方法使之返回true如果你需要合并操作符在只有一个运算元的时候被调起。一个使用例子就是如果你用合并操作来基于TTL修改数值，这样他就会在压缩之后被删除（或者用一个压缩过滤器）

## JSON例子

使用我们的通用MergeOperator接口，我们现在有能力实现我们的json例子了。

```cpp
    // A 'model' pseudo-code merge operator with json update semantics
    // We pretend we have some in-memory data-structure (called JsonDataStructure) for
    // parsing and serializing json strings.
    class JsonMergeOperator : public MergeOperator {          // not associative
     public:
      virtual bool FullMerge(const Slice& key,
                             const Slice* existing_value,
                             const std::deque<std::string>& operand_list,
                             std::string* new_value,
                             Logger* logger) const override {
        JsonDataStructure obj;
        if (existing_value) {
          obj.ParseFrom(existing_value->ToString());
        }

        if (obj.IsInvalid()) {
          Log(logger, "Invalid json string after parsing: %s", existing_value->ToString().c_str());
          return false;
        }

        for (const auto& value : operand_list) {
          auto split_vector = Split(value, " = ");      // "xyz[0] = 5" might return ["xyz[0]", 5] as an std::vector, etc.
          obj.SelectFromHierarchy(split_vector[0]) = split_vector[1];
          if (obj.IsInvalid()) {
            Log(logger, "Invalid json after parsing operand: %s", value.c_str());
            return false;
          }
        }

        obj.SerializeTo(new_value);
        return true;
      }


      // Partial-merge two operands if and only if the two operands
      // both update the same value. If so, take the "later" operand.
      virtual bool PartialMerge(const Slice& key,
                                const Slice& left_operand,
                                const Slice& right_operand,
                                std::string* new_value,
                                Logger* logger) const override {
        auto split_vector1 = Split(left_operand, " = ");   // "xyz[0] = 5" might return ["xyz[0]", 5] as an std::vector, etc.
        auto split_vector2 = Split(right_operand, " = ");

        // If the two operations update the same value, just take the later one.
        if (split_vector1[0] == split_vector2[0]) {
          new_value->assign(right_operand.data(), right_operand.size());
          return true;
        } else {
          return false;
        }
      }

      virtual const char* Name() const override {
        return "JsonMergeOperator";
       }
    };

    ...

    // How to use it
    DB* dbp;
    Options options;
    options.merge_operator.reset(new JsonMergeOperator);
    DB::Open(options, "/tmp/db", &dbp);
    std::shared_ptr<DB> db_(dbp);
    ...
    // Put/store the json string into to the database
    db_->Put(put_option_, "json_obj_key",
             "{ employees: [ {first_name: john, last_name: doe}, {first_name: adam, last_name: smith}] }");

    ...

    // Use the "merge operator" to incrementally update the value of the json string
    db_->Merge(merge_option_, "json_obj_key", "employees[1].first_name = lucy");
    db_->Merge(merge_option_, "json_obj_key", "employees[0].last_name = dow");

```


# 错误处理

如果MergeOperator::PartialMerge()返回false，意味着这个合并需要被推迟（入栈）直到我们遇到一个Put/Delete数值来进行FullMerge。然而，如果FullMerge返回false，这就会被认为是“中断”错误。这意味着RocksDB会给客户返回一个Status::Corruption之类的消息。因此， MergeOperator::FullMerge() 应该在只有客户绝对无法处理这个错误的时候返回false。

对于AssociativeMergeOperator，Merge方法使用跟 MergeOperator::FullMerge() 一样的错误处理逻辑。只有在没有办法从逻辑上处理这个错误的时候返回false。在上面的计数器例子，我们的Merge方法总是返回true，因为我们会把任何错误的值视为0。

# 检查以及最佳实践

总的来说，我们已经描述了合并操作，以及如何用它。这里有一些提示，关于什么时候，如何使用MergeOperator和AssociativeMergeOperator。

## 什么时候用merge

如果下面为真：

- 你的数据需要增量更新
- 你需要在知道新数据之前读数据

那么使用上面说的两个合并操作符吧。

## 结合数据

如果下面是真：

- 如果你的合并运算元根你Put的值有一样的格式
- 把多个运算元组合成一个运算元是ok的（只要组合的顺序正确）

那么，就用 **AssociativeMergeOperator**

## 通用合并

如果上面的两条有一条不满足，那么使用 **MergeOperator**。

如果某些时候可以把多个运算元合并成一个（但不总是可以）：
- 使用MergeOpertaor
- 必要时让PartialMerge返回true

## Tips

多路复用：RocksDB DB对象在构造的时候只能传入一个合并操作符，你的用户定义的合并操作类可以根据传入的数据进行不同的处理。key和value本身，会被传递给合并运算符；所以大家可以在运算元里面实现不同的运算方法，然后在MergeOperator里面各自使用不同的函数。

我的用例是不是符合结合律？：如果你不确定是不是能满足“结合律”，你总是可以使用MergeOperator的。AssociativeMergeOperator是MergeOperator的直接子类，所以任何使用AssociativeMergeOperator可以解决的问题，在通用MergeOperator都可以解决。AssociativeMergeOperator只是为了提供便利性。

## 有用的链接

[合并+压缩实现细节](https://rocksdb.org.cn/doc/Merge-Operator-Implementation.html) 为那些希望了解合并操作符对他们的代码的影响的RocksDb工程师提供。

