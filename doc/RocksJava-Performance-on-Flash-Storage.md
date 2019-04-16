在四月14日，我们开始为RocksDB构建一个[Java拓展](RocksJava-Basics.md)。这一页展示RocksJava在Flash存储上的性能测试结果。C++的性能测试结果可以在[这里]()找到

# 设置

所有的性能测试都在同一个机器上进行，下面是测试环境的配置：

-	测试10亿个键值对。每个key 16 bytes，每个值800 bytes。总数据库裸数据大小大于1TB.
-	Intel(R) Xeon(R) CPU E5-2660 v2 @ 2.20GHz, 40 cores.
-	25 MB CPU cache, 144 GB Ram
-	CentOS release 5.2 (Final).
-	实验运行于 Funtoo chroot 环境.
-	g++ (Funtoo 4.8.1-r2) 4.8.1
-	Java(TM) SE Runtime Environment (build 1.7.0_55-b13)
-	Java HotSpot(TM) 64-Bit Server VM (build 24.55-b03, mixed mode)
-	版本 85f9bb4 被包括进来.
-	1G rocksdb block cache.
-	Snappy 1.1.1 为配置的压缩（compression）算法.
-	JEMALLOC不开启.

# 批量顺序加载key(Test2)

这个测试考量使用RocksJava加载10亿个key到数据库的性能。key被按顺序插入。数据库在开始的时候为空的，然后渐渐被填满。加载过程中无读取。下面是RocksJava批量加载的性能：

```
fillseq          :     2.48233 micros/op;  311.2 MB/s; 1000000000 ops done;  1 / 1 task(s) finished.
```

跟我们在[Rocksdb C++性能测试]()的时候类似，RocksJava被配置使用多线程压缩，所以多个线程可以同步压缩多个层的没有交叉的键范围。我们的结果显示，RocksJava可以做大 300+MB/s，或者400K写/s的写入性能。

这里是使用RocksJava进行批量加载的命令。

```
bpl=10485760;overlap=10;mcz=0;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864;wbs=134217728; dds=0; sync=false; t=1; vs=800; bs=65536; cs=1048576; of=500000; si=1000000;
./jdb_bench.sh  --benchmarks=fillseq  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --threads=$t  --key_size=10  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --compression_type=snappy  --cache_numshardbits=4  --open_files=$of  --verify_checksum=true  --db=/rocksdb-bench/java/b2  --sync=$sync  --disable_wal=true  --stats_interval=$si  --compression_ratio=0.50  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=false  --cache_remove_scan_count_limit=16 --num=1000000000
```

# 随机读（Test4）

这个性能测试测量RocksJava在10亿个key的情况下的随机读性能，每个key 16 byte，值800 byte。在这个测试中，RocksJava配置为每个块4KB，snappy压缩被打开。应用有32个线程在对数据库进行随机读。另外，RocksJava被配置为检测每个读的校验和。

这个性能测试分两步进行。第一步，数据先使用线性写入加载所有10亿个key到数据库。一旦加载结束，他进入第二部分，这里32个线程会同步发起随机读请求。我们只测量第二部分的性能。

```
readrandom       :     7.67180 micros/op;  101.4 MB/s; 1000000000 / 1000000000 found;  32 / 32 task(s) finished.
```

我们的结果显示，RocksJava可以做到100+ MB/s，或者130K次/s的读请求

这里是用来做性能测试的命令：

```
echo "Load 1B keys sequentially into database....."
n=1000000000; r=1000000000; bpl=10485760;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864;wbs=134217728; dds=1; sync=false; t=1; vs=800; bs=4096; cs=1048576; of=500000; si=1000000;
./jdb_bench.sh  --benchmarks=fillseq  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$n  --threads=$t  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=/rocksdb-bench/java/b4  --sync=$sync  --disable_wal=true  --compression_type=snappy  --stats_interval=$si  --compression_ratio=0.50  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=false

echo "Reading 1B keys in database in random order...."
bpl=10485760;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864; wbs=134217728; dds=0; sync=false; t=32; vs=800; bs=4096; cs=1048576; of=500000; si=1000000;
./jdb_bench.sh  --benchmarks=readrandom  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$n  --reads=$r  --threads=$t  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=/rocksdb-bench/java/b4  --sync=$sync  --disable_wal=true  --compression_type=none  --stats_interval=$si  --compression_ratio=0.50  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=true
```

# 多线程读和单线程写(Test5)

这个性能测试测量从10亿个key中随机读取1亿个key，同时会并发执行更新操作。这个实验的读取次数被限制为1亿以减少测量时间。类似我们做的test4，每个key还是16 byte，value为800 byte，RocksJava被配置为一个块4Kb，并且snappy压缩打开。在这个测试，有32个读线程，一个独立的随机写线程，写入每秒10k个。下面是随机读和写的结果：

```
readwhilewriting :     9.55882 micros/op;   81.4 MB/s; 100000000 / 100000000 found;  32 / 32 task(s) finished.

```

结果显示，同步执行更新的时候，读取速度大概80MB/s，或者每秒100k次读取。

下面是用于测试的命令：

```
echo "Load 1B keys sequentially into database....."
dir="/rocksdb-bench/java/b5"
num=1000000000; r=100000000;  bpl=536870912;  mb=67108864;  overlap=10;  mcz=2;  del=300000000;  levels=6;  ctrig=4; delay=8;  stop=12;  wbn=3;  mbc=20;  wbs=134217728;  dds=false;  sync=false;  vs=800;  bs=4096;  cs=17179869184; of=500000;  wps=0;  si=10000000;
./jdb_bench.sh  --benchmarks=fillseq  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$num  --threads=1  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=$dir  --sync=$sync  --disable_wal=true  --compression_type=snappy  --stats_interval=$si  --compression_ratio=0.5  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=false
echo "Reading while writing 100M keys in database in random order...."
bpl=536870912;mb=67108864;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4;delay=8;stop=12;wbn=3;mbc=20;wbs=134217728;dds=false;sync=false;t=32;vs=800;bs=4096;cs=17179869184;of=500000;wps=10000;si=10000000;
./jdb_bench.sh  --benchmarks=readwhilewriting  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$num  --reads=$r  --writes_per_second=10000  --threads=$t  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=$dir  --sync=$sync  --disable_wal=false  --compression_type=snappy  --stats_interval=$si  --compression_ratio=0.5  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=true  --writes_per_second=$wps

```


