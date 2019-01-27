# SeekForPrev API

从4.13开始，RocksDB新加了一个Iterator::SeekForPrev()调用。这个新的API与seek不同，允许查找小于或者等于目标key的最后一个key。

```cpp
// Suppose we have keys "a1", "a3", "b1", "b2", "c2", "c4".
auto iter = db->NewIterator(ReadOptions());
iter->Seek("a1");        // iter->Key() == "a1";
iter->Seek("a3");        // iter->Key() == "a3";
iter->SeekForPrev("c4"); // iter->Key() == "c4";
iter->SeekForPrev("c3"); // iter->Key() == "c2";

```

SeekForPrev的行为基本就是如下：

```cpp
Seek(target); 
if (!Valid()) {
  SeekToLast();
} else if (key() > target) { 
  Prev(); 
}
```

事实上，这个API会做更多的事情。其中一个例子就是，加入我们有key："a1", "a3", "a5", "b1"，然后我们打开了prefix_extractor，使用第一个byte做前缀。如果我们希望找到小于等于"a6"的最后一个key。上面的代码不使用SeekForPrev，是不能正确给出结果的。由于在Seek("a6")之后Prev()，迭代器会进入一种无效状态。但是现在，你只需要调用SeekForPrev("a6")就能得到"a5"

同时，SeekforPrev API在前缀模式对Prev操作提供内部支持。现在，Next和Prev可以在前缀模式混用了。感谢SeekForPrev

