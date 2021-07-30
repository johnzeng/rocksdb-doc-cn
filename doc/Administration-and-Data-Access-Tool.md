# Ldb 工具

ldb命令行工具提供多种数据访问和数据库管理命令。下面列出了一些样例。如果需要更多帮助信息，请直接不带参数运行ldb工具，或者运行tools/ldb_test.py内的单元测试。

数据访问样例：

```shell
    $./ldb --db=/tmp/test_db --create_if_missing put a1 b1
    OK 

    $ ./ldb --db=/tmp/test_db get a1
    b1
 
    $ ./ldb --db=/tmp/test_db get a2
    Failed: NotFound:

    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
 
    $ ./ldb --db=/tmp/test_db scan --hex
    0x6131 : 0x6231
 
    $ ./ldb --db=/tmp/test_db put --key_hex 0x6132 b2
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
    a2 : b2
 
    $ ./ldb --db=/tmp/test_db get --value_hex a2
    0x6232
 
    $ ./ldb --db=/tmp/test_db get --hex 0x6131
    0x6231
 
    $ ./ldb --db=/tmp/test_db batchput a3 b3 a4 b4
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
    a2 : b2
    a3 : b3
    a4 : b4
 
    $ ./ldb --db=/tmp/test_db batchput "multiple words key" "multiple words value"
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    Created bg thread 0x7f4a1dbff700
    a1 : b1
    a2 : b2
    a3 : b3
    a4 : b4
    multiple words key : multiple words value

```

以十六进制导出已经存在的rocksdb库:

```shell
$ ./ldb --db=/tmp/test_db dump --hex > /tmp/dbdump
```

加载十六进制格式的数据进新的rocksdbdb库

```shell
$ cat /tmp/dbdump | ./ldb --db=/tmp/test_db_new load --hex --compression_type=bzip2 --block_size=65536 --create_if_missing --disable_wal
```

压缩一个已经存在的rocksdb库

```shell
$ ./ldb --db=/tmp/test_db_new compact --compression_type=bzip2 --block_size=65536
```

你可以通过--column_family=<string>来指定你要查询的column family。

--try_load_options 将会尝试加载配置文件来打开数据库。如果你总是要用这个配置去打开数据库，这是一个好方法。如果你使用默认配置打开数据库，它可能破坏LSM-Tree结构并导致无法自动回复。

# SST dump tool

sst_dump工具能用来获取一个指定的SST文件视图。对一个SST文件sst_dump有多种操作功能。

```shell
$ ./sst_dump
file or directory must be specified.

sst_dump --file=<data_dir_OR_sst_file> [--command=check|scan|raw]
    --file=<data_dir_OR_sst_file>
      Path to SST file or directory containing SST files

    --command=check|scan|raw|verify
        check: Iterate over entries in files but dont print anything except if an error is encounterd (default command)
        scan: Iterate over entries in files and print them to screen
        raw: Dump all the table contents to <file_name>_dump.txt
        verify: Iterate all the blocks in files verifying checksum to detect possible coruption but dont print anything except if a corruption is encountered
        recompress: reports the SST file size if recompressed with different
                    compression types

    --output_hex
      Can be combined with scan command to print the keys and values in Hex

    --from=<user_key>
      Key to start reading from when executing check|scan

    --to=<user_key>
      Key to stop reading at when executing check|scan

    --prefix=<user_key>
      Returns all keys with this prefix when executing check|scan
      Cannot be used in conjunction with --from

    --read_num=<num>
      Maximum number of entries to read when executing check|scan

    --verify_checksum
      Verify file checksum when executing check|scan

    --input_key_hex
      Can be combined with --from and --to to indicate that these values are encoded in Hex

    --show_properties
      Print table properties after iterating over the file when executing
      check|scan|raw

    --set_block_size=<block_size>
      Can be combined with --command=recompress to set the block size that will
      be used when trying different compression algorithms

    --compression_types=<comma-separated list of CompressionType members, e.g.,
      kSnappyCompression>
      Can be combined with --command=recompress to run recompression for this
      list of compression types

    --parse_internal_key=<0xKEY>
      Convenience option to parse an internal key on the command line. Dumps the
      internal key in hex format {'key' @ SN: type}
```

### 导出SST文件块

```shell
./sst_dump --file=/path/to/sst/000829.sst --command=raw
```

这个命令会产生一个名为 /path/to/sst/000829_dump.txt. 的txt文件。这个文件会包含所有的十六进制编码的索引块和数据块，同时包含表配置，footer细节和原数据索引细节。

### 打印SST文件条目

```shell
./sst_dump --file=/path/to/sst/000829.sst --command=scan --read_num=5
```

这个命令会打印SST文件内头5个key。输出可能像这样：

```shell
'Key1' @ 5: 1 => Value1
'Key2' @ 2: 1 => Value2
'Key3' @ 4: 1 => Value3
'Key4' @ 3: 1 => Value4
'Key5' @ 1: 1 => Value5
```

输出能被这样解释

```
'<key>' @ <sequence number>: <type> => <value>
```

请注意如果你的key是非Ascii编码的，它很难打印在屏幕上，这种情况下使用--output_hex是个好方法

```
./sst_dump --file=/path/to/sst/000829.sst --command=scan --read_num=5 --output_hex
```

你也可以使用--from和--to指定开始和结束位置

```shell
./sst_dump --file=/path/to/sst/000829.sst --command=scan --from="key2" --to="key4"
```

使用--input_key_hex选项传入十六进制--from和--to

```shell
./sst_dump --file=/path/to/sst/000829.sst --command=scan --from="0x6B657932" --to="0x6B657934" --input_key_hex
```

### 检查SST文件

```shell
./sst_dump --file=/path/to/sst/000829.sst --command=check --verify_checksum
```

这个命令会迭代SST文件内的所有条目，但是只会在出错时打印信息。它也会校验文件。

##### 打印 SST file 配置

```
./sst_dump --file=/path/to/sst/000829.sst --show_properties
```

这个命令会读取SST文件配置并打印，输出像这样

```
from [] to []
Process /path/to/sst/000829.sst
Sst file format: block-based
Table Properties:
------------------------------
  # data blocks: 26541
  # entries: 2283572
  raw key size: 264639191
  raw average key size: 115.888262
  raw value size: 26378342
  raw average value size: 11.551351
  data block size: 67110160
  index block size: 3620969
  filter block size: 0
  (estimated) table size: 70731129
  filter policy name: N/A
  # deleted keys: 571272
```

尝试不同的压缩算法

sst_dump能够被用来测试文件在不同压缩算法下的大小

```
./sst_dump --file=/path/to/sst/000829.sst --show_compression_sizes
```

通过使用 --show_compression_sizes sst_dump 能够在内存中用不同的算法重建SST文件，并输出其大小，输出像这样

```shell
from [] to []
Process /path/to/sst/000829.sst
Sst file format: block-based
Block Size: 16384
Compression: kNoCompression Size: 103974700
Compression: kSnappyCompression Size: 103906223
Compression: kZlibCompression Size: 80602892
Compression: kBZip2Compression Size: 76250777
Compression: kLZ4Compression Size: 103905572
Compression: kLZ4HCCompression Size: 97234828
Compression: kZSTDNotFinalCompression Size: 79821573
```

这些文件被创建在内容中，并且他们的块大小为16KB，块大小能通过--set_block_size改变。