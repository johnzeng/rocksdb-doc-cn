#String选项以及Map选项

用户通过Options类向RocksDB传递选项。除了通过使用Options类设置选项，还可以使用另外两(写作两，读作三)个方法设置：

- 从一个选项文件生成一个选项类
- 通过一个选项字符串获得
- 通过一个字符串map获得

## 选项字符串

可以通过调用helper函数GetColumnFamilyOptionsFromString或者GetDBOptionsFromString，将一个带有配置信息的字符串传入，来构造选项。另外还有一个特别的函数：GetBlockBasedTableOptionsFromString和GetPlainTableOptionsFromString来获得表设置选项。

一个选项字符串如下：

table_factory=PlainTable;prefix_extractor=rocksdb.CappedPrefix.13;comparator=leveldb.BytewiseComparator;compression_per_level=kBZip2Compression:kBZip2Compression:kBZip2Compression:kNoCompression:kZlibCompression:kBZip2Compression:kSnappyCompression;max_bytes_for_level_base=986;bloom_locality=8016;target_file_size_base=4294976376;memtable_huge_page_size=2557;max_successive_merges=5497;max_sequential_skip_in_iterations=4294971408;arena_block_size=1893;target_file_size_multiplier=35;min_write_buffer_number_to_merge=9;max_write_buffer_number=84;write_buffer_size=1653;max_compaction_bytes=64;max_bytes_for_level_multiplier=60;memtable_factory=SkipListFactory;compression=kNoCompression;bottommost_compression=kDisableCompressionOption;min_partial_merge_operands=7576;level0_stop_writes_trigger=33;num_levels=99;level0_slowdown_writes_trigger=22;level0_file_num_compaction_trigger=14;compaction_filter=urxcqstuwnCompactionFilter;soft_rate_limit=530.615385;soft_pending_compaction_bytes_limit=0;max_write_buffer_number_to_maintain=84;verify_checksums_in_compaction=false;merge_operator=aabcxehazrMergeOperator;memtable_prefix_bloom_size_ratio=0.4642;memtable_insert_with_hint_prefix_extractor=rocksdb.CappedPrefix.13;paranoid_file_checks=true;force_consistency_checks=true;inplace_update_num_locks=7429;optimize_filters_for_hits=false;level_compaction_dynamic_level_bytes=false;inplace_update_support=false;compaction_style=kCompactionStyleFIFO;purge_redundant_kvs_while_flush=true;hard_pending_compaction_bytes_limit=0;disable_auto_compactions=false;report_bg_io_stats=true;compaction_filter_factory=mpudlojcujCompactionFilterFactory;

每个选项都是通过 <选项名>:<选项值>，然后使用分号;进行分割。支持的选项，可以查看下面的内容。

## 选项map

类似的，用户可以通过一个string map来构造选项类，通过调用GetColumnFamilyOptionsFromMap，GetDBOptionsFromMap，GetBlockBasedTableOptionsFromMap或者GetPlainTableOptionsFromMap即可。将string到string的map传入，从字符选项名映射到字符选项值。一个字符map的选项如下：

```cpp
  std::unordered_map<std::string, std::string> cf_options_map = {
      {"write_buffer_size", "1"},
      {"max_write_buffer_number", "2"},
      {"min_write_buffer_number_to_merge", "3"},
      {"max_write_buffer_number_to_maintain", "99"},
      {"compression", "kSnappyCompression"},
      {"compression_per_level",
       "kNoCompression:"
       "kSnappyCompression:"
       "kZlibCompression:"
       "kBZip2Compression:"
       "kLZ4Compression:"
       "kLZ4HCCompression:"
       "kXpressCompression:"
       "kZSTD:"
       "kZSTDNotFinalCompression"},
      {"bottommost_compression", "kLZ4Compression"},
      {"compression_opts", "4:5:6:7"},
      {"num_levels", "8"},
      {"level0_file_num_compaction_trigger", "8"},
      {"level0_slowdown_writes_trigger", "9"},
      {"level0_stop_writes_trigger", "10"},
      {"target_file_size_base", "12"},
      {"target_file_size_multiplier", "13"},
      {"max_bytes_for_level_base", "14"},
      {"level_compaction_dynamic_level_bytes", "true"},
      {"max_bytes_for_level_multiplier", "15.0"},
      {"max_bytes_for_level_multiplier_additional", "16:17:18"},
      {"max_compaction_bytes", "21"},
      {"soft_rate_limit", "1.1"},
      {"hard_rate_limit", "2.1"},
      {"hard_pending_compaction_bytes_limit", "211"},
      {"arena_block_size", "22"},
      {"disable_auto_compactions", "true"},
      {"compaction_style", "kCompactionStyleLevel"},
      {"verify_checksums_in_compaction", "false"},
      {"compaction_options_fifo", "23"},
      {"max_sequential_skip_in_iterations", "24"},
      {"inplace_update_support", "true"},
      {"report_bg_io_stats", "true"},
      {"compaction_measure_io_stats", "false"},
      {"inplace_update_num_locks", "25"},
      {"memtable_prefix_bloom_size_ratio", "0.26"},
      {"memtable_huge_page_size", "28"},
      {"bloom_locality", "29"},
      {"max_successive_merges", "30"},
      {"min_partial_merge_operands", "31"},
      {"prefix_extractor", "fixed:31"},
      {"optimize_filters_for_hits", "true"},
  };
```

具体支持的选项，可以参考下面的章节。

## 如何找到字符串选项和map选项支持的选项

不管是字符串还是map配置，选项名映射到目标类（如DBOptions， ColumnFamilyOptions， BlockBasedTableOptions，或者PlainTableOptions）的变量名。对于DBOptions和ColumnFamilyOptions，你可以在 [options.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h) 找到具体的版本，然后找到他们支持的选项列表和对应的类。另外两个选项，在 [table.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/table.h) 文件里面。

主意，尽管类里面的多数选项都是支持的，例外还是有的。你需要通过[util/options_helper.h](https://github.com/facebook/rocksdb/blob/master/util/options_helper.h) 文件里的db_options_type_info，cf_options_type_info和block_based_table_type_info找到对应版本的所有支持的选项。

如果某个选项是一个回调函数，例如 comparators，compaction filter，以及merge options，你需要把回调类的指针传入，通常还需要进行一定的类型转换。

也有例外，某些回调类支持通过字符选项或者map选项传入，如：

- Prefix extractor(选项名prefix_extractor)，他的值可以设置为rocksdb.FixedPrefix.<前缀长度> 或者 rocksdb.CappedPrefix.<前缀长度>。
- Filter policy(选项名filter_poilcy)，他的值可以设置为bloomfilter:<bits_per_key>:<use_block_based>
- Table factory (选项名 table_factory)。他的值是BlockBasedTable 或者 PlainTable。除此之外，两个特殊的选项字符串名可以用于设置这个选项，也就是block_based_table_factory或者plain_table_factory。这两个选项的值可以是BlockBasedTableOptions或者PlainTableOptions。
- Memtable Factory（选项名memtable_factory）。它可以是skip_list，prefix_hash，hash_linkedlist，vector或者cuckoo。

