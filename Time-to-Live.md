## RocksDB可以打开生存时间(TTL)支持

# 使用场景

这个API可以用于插入一个KV对的时候就需要他们在一个不那么严格的TTL时间内被删除的场景，他保证插入的键值对会在db里保存至少ttl时间，然后db会尽最大努力，在炒锅ttl时间后尽快删除他们。

# 行为

- TTL单位为秒
- (int32_t)Timestamp(creation)在调用Put的时候会在内部被追加到value的后面
- TTL值的过期操作只在压缩过程中触发：（Timestamp+ttl<time_now）
- Get/Iterator可能会返回已经过期的键（他们还没发生压缩）
- 不同的Open的TTL可能不同
- 例子：在t=0时打开Open1，ttl=4，然后插入k1，k2，在t=2关闭。t＝3的时候打开Open2，ttl＝3。现在k1,k2会在 t>=5的时候被删除
- 打开的时候如果read_only=true会用通常的只读模式打开。压缩器不会被触发（不管是手动的还是自动的），所以没有过期项被移除。

# 限制

不声明或者传入非正的TTL会使其其行为等同于TTL=无限

# !!警告!!

- 直接调用DB::Open来重新打开一个通过这个API创建的数据库会得到一个错误的值（有timestamp在最后追加），并且这次打开没有ttl效果，所以请确保总是用这个API打开db
- 如果传入一个小的正数TTL，请小心，因为整个数据库都会快速的被删除。

# API

定义于 <rocksdb/utilities/db_ttl.h>

```cpp
static Status DBWithTTL::Open(const Options& options, const std::string& name, StackableDB** dbptr, int32_t ttl = 0, bool read_only = false);
```



