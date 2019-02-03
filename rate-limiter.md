使用RocksDB的时候，用户可能因为许多原因，希望控制最大写速度在一个范围。例如，闪存写的时候如果超过特定阀值，会引发严重的读延迟峰值。因为你已经在读这篇文章，我相信你已经知道为什么你需要一个限流器。事实上，RocksDB自带一个[限流器](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/rate_limiter.h)，在大多数场景，应该都是够用了。

# 如何使用

通过调用NewGenericRateLimiter创建一个RateLimiter对象，可以对每个RocksDB实例分别创建，或者在RocksDB实例之间共享，以此控制落盘和压缩的写速率总量。

```cpp
RateLimiter* rate_limiter = NewGenericRateLimiter(
    rate_bytes_per_sec /* int64_t */, 
    refill_period_us /* int64_t */,
    fairness /* int32_t */);
```

参数：

- rate_bytes_per_sec：通常这是唯一一个你需要关心的参数。他控制压缩和落盘的总速率，单位为bytes/秒。现在，RocksDB并不强制限制除了落盘和压缩以外的操作（如写WAL）
- refill_period_us：这个控制令牌被填充的频率。例如，当rate_bytes_per_sec被设置为10MB/s然后refill_period_us被设置为100ms，那么就每100ms会从新填充1MB的限量。更大的数值会导致突发写，而更小的数值会导致CPU过载。默认数值100,000应该在大多数场景都能很好工作了
- fairness：RateLimiter接受高优先级请求和低优先级请求。一个低优先级任务会被高优先级任务挡住。现在，RocksDB把来自压缩的请求认为是低优先级的，把来自落盘的任务认为是高优先级的。如果落盘请求不断地过来，低优先级请求会被拦截。这个fairness参数保证低优先级请求，在即使有高优先级任务的时候，也会有1/fairness的机会被执行，以避免低优先级任务的饿死。默认是10通常是可以的。

尽管令牌会以refill_period_us设定的时间按照间隔来填充，我们仍然需要保证一次写爆发中的最大字节数，因为我们不希望看到令牌堆积了很久，然后在一次写爆发中一次性消耗光，这显然不符合我们的需求。GetSingleBurstBytes会返回这个令牌数量的上限。

这样，每次写请求前，都需要申请令牌。如果这个请求无法被满足，请求会被阻塞，直到令牌被填充到足够完成请求。比如：

```cpp
// block if tokens are not enough
rate_limiter->Request(1024 /* bytes */, rocksdb::Env::IO_HIGH); 
Status s = db->Flush();
```

如果有需要，用户还可以通过SetBytesPerSecond动态修改限流器每秒流量。参考[include/rocksdb/rate_limiter.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/rate_limiter.h) 了解更多细节

# 定制

对那些RocksDB提供的原生限流器无法满足需求的用户，他们可以通过继承[include/rocksdb/rate_limiter.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/rate_limiter.h) 来实现自己的限流器。

