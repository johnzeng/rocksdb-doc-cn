# 事件监听器

事件监听器EventListener类包含一系列的回调函数，当特定的RocksDB事件发生时，比如一次落盘或者压缩工作结束，就会被调用。这些回调接口可以被用于开发一些自定义功能，比如统计信息收集或者外部压缩算法。可用的监听器回调可以再[include/rocksdb/listener.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/listener.h)中找到

## 如何使用？

在ColumnFamilyOptions中，有一系列的被调用监听器，允许开发者加入自定义的监听器来监听特定rocksdb实例或者一个列族中的事件。

```cpp
// A vector of EventListeners which call-back functions will be called
// when specific RocksDB event happens.
std::vector<std::shared_ptr<EventListener>> listeners;
```

如果详见听一个rocksdb实例或者一个列族，可以通过简单增加自定义的EventListener到ColumnFamilyOptions::listeners，然后使用这个options来打开DB。

```cpp
// listen to a column family of a rocksdb instance
ColumnFamilyOptions cf_options;
...
cf_options.listeners.emplace_back(new MyListener());
```

如果希望监听一个实例的多个列族，你可以对每个列族使用独立的事件监听实例。

```cpp
// one listener for each column family
for (size_t i = 0; i < cf_options.size(); ++i) {
  cf_options[i].listeners.emplace_back(new MyListener());
}
```

或者对所有的列族使用同样的监听器实例：

```cpp
// one same listener for all column families.
EventListener my_listener = new MyListener();
for (size_t i = 0; i < cf_options.size(); ++i) {
  cf_options[i].listeners.emplace_back(my_listener);
}
```

注意，在所有的例子，除非在这个文档特别声明，所有的EventListener回调函数必须以一种线程安全的形式开发，不管这个EventListener是不是只给一个列族使用（例如，想象一下OnCompactionCompleted的使用例子，他可以被一个列族的多个线程调用，因为一个列族可以同一时间完成多个压缩任务）

## 监听特定时间

所有的EventListener的默认行为都是无操作。这允许开发者只关注他们关心的事情。为了监听一个特定事件，只需要实现相关的回调接口即可。例如，下面的EventListener计算从DB打开开始，落盘工作结束的次数，只需要实现OnFlushCompleted即可：

```cpp
class FlushCountListener : public EventListener {
 public:
  FlushCountListener() : flush_count_(0) {}
  void OnFlushCompleted(
      DB* db, const std::string& name,
      const std::string& file_path,
      bool triggered_writes_slowdown,
      bool triggered_writes_stop) override {
    flush_count_++;
  }
 private:
  std::atomic_int flush_count_;
}; 
```

## 多线程

所有的EventListener回调会被事件发生的线程调用。例如，这里有一个RocksDB后台落盘线程，他做完实际的落盘工作之后就调用 EventListener::OnFlushCompleted()。这允许开发者从EventListener通过线程本地的数据收集线程独立的统计数据。

## 上锁

所有的EventListener都被设计为在没有持有任何线程DB互斥锁的情况下调用。这是为了防止 在使用复杂的EventListener回调的时候 潜在的死锁和性能问题。然而，所有的EventListener回调函数都不应该花费太多的时间，否则RocksDB可能会被上锁。例如，在EventListener回调中，不建议做DB::CompactFiles() （因为这会运行挺长一段时间）或者在一个线程发起非常多的DB::Put()（因为Put可能在某些场合会block）。然而，在不运行EventListener回调的线程执行DB::CompactFiles()和DB::Put()是安全的。


