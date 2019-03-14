RocksJava是为了给RocksDB构建一个高性能，但是易用的java驱动的工程。

RocksJava由3层构成：

- org.rocksdb包里面的Java类，构成RocksJava API。Java用户只会直接接触到这一层。
- C++的JNI代码，提供Java API和Rock是DB之间的链接。
- C++层的RocksDB本身，并且编译成了一个native库，被JNI层使用。

我们尽力是RocksJava的API和RocksDB的c++ API同步，但是他经常会落后。我们高度鼓励社区贡献代码。。。如果你需要某个特定的API在c++有但是Java没有，提PR吧

在这一页，你会学习RocksDB Java API的基础。

# 开始

你可以使用我们发布的预编译好的Maven包，或者自己从代码构建RocksJava。

## Maven

我们在Maven Central发布了RocksJava，这样你就可以依赖jar包而不是自己构建了 [https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.rocksdb%22](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.rocksdb%22)

我们同时发布了通用jar包（rocksdbjni-X.X.X.jar），包含了所有支持的平台的native库和java class文件，也发布了一些小平台的jar包（例如rocksdbjni-X.X.X-linux65.jar）

在一个支持Maven风格依赖的构建系统中，最简单的使用RocksJava的方法就是增加一个RocksJava的依赖，例如，如果你是用Maven：

```
<dependency>
  <groupId>org.rocksdb</groupId>
  <artifactId>rocksdbjni</artifactId>
  <version>5.5.1</version>
</dependency>
```

**给Windows用户的备注：**如果你正在MS的Windows上使用Maven Central编译的包，他们使用Microsoft Visual Studio 2015编译的，如果你没有安装“Microsoft Visual C++ 2015 Redistributable”,那么你需要从[https://www.microsoft.com/en-us/download/details.aspx?id=48145](https://www.microsoft.com/en-us/download/details.aspx?id=48145)安装他们，或者你需要自己从源码编译（rocksdb）

## 从源代码编译

要编译RocksJava，你首先需要设置你的JAVA_HOME环境变量，指向你安装的java SDK目录（必须Java 1.7+）。你必须有预编译好的RocksDB的native库，参考[INSTALL.md](https://github.com/facebook/rocksdb/blob/master/INSTALL.md)。一旦JAVA_HOME正确设置，并且你安装了需要的库，只要通过以下命令就可以构建rocksdbjava：

```
$ make -j8 rocksdbjava
```

这会生成rocksdbjni.jar和librocksdbjni.so（或者如果你是macOS，librocksdbjni.jnilib），位置在rocksdb根目录的java/target目录。特别的，rocksdbjni.jar包含Java类，定义了rocksdb的Java API，而librocksdbjni.so包含C++ rocksdb库和rocksdbjni.jar中定义的java类的native实现。

如果希望运行单元测试：

```
$ make jtest
```

清理：

```
$ make jclean
```

# 样例

我们在[这里](https://github.com/facebook/rocksdb/tree/master/java/samples/src/main/java) 提供了一些使用样例，你可以直接参考代码

# 内存管理

RocksJava中的许多Java对象后面是C++对象，Java的对象拥有对应的控制权。由于C++没有跟Java一样的自动垃圾回收的概念，我们必须在使用完毕之后，显式释放C++对象使用的内存。

任何RocksJava中的管理了C++对象的Java对象会继承自org.rocksdb.AbstractNativeReference，当你用完这个对象后，这个父类被用来帮助管理和清理他手上的所有C++对象。有两个机制：

## AbstractNativeReference#close()

当用户使用完RocksJava的一个对象后，这个方法应该被用户显式调用。如果C++对象被分配而没有被释放，那么他们会在第一次调用这个方法的时候被释放。

为了简化使用，这个方法重载了java.lang.AutoCloseable#close()，这就允许他使用ARM (Automatic Resource Management自动资源管理)风格的构造方法，例如java SE 7的 [try-with-resources](try-with-resources)声明

## AbstractNativeReference#finalize()

当一个对象的所有存储引用都失效，并且对象就要进行垃圾回收的时候，这个方法被Java的Finalizer线程调用。他最后会委托给AbstractNativeReference#close()。不过，用户不应该依赖他，而应该认为这个是一个最后的防线。

他保证了Java对象手头的C++对象最终会被回收。但是他不能帮助RocksJava管理所有内存，因为native C++对象的内存在C++的堆上分配，然后返回给Java对象，这些对Java的GC机制是不可见的，所以JVM无法正确计算GC的内存压力。 使用完一个对象之后，**用户总是应该显式调用AbstractNativeReference#close()**。

# 打开数据库

一个rocksdb数据库有一个名字，对应于文件系统上的一个文件夹。该数据库所有的数据都会存储在这个文件夹中。下面的例子展示如何打开一个数据库，如果有需要，自动创建：

```java
import org.rocksdb.RocksDB;
import org.rocksdb.Options;
...
  // a static method that loads the RocksDB C++ library.
  RocksDB.loadLibrary();

  // the Options class contains a set of configurable DB options
  // that determines the behaviour of the database.
  try (final Options options = new Options().setCreateIfMissing(true)) {
    
    // a factory method that returns a RocksDB instance
    try (final RocksDB db = RocksDB.open(options, "path/to/db")) {
    
        // do something
    }
  } catch (RocksDBException e) {
    // do some error handling
    ...
  }
...
```

*TIP: 你可能注意到上面的RocksDBException类。这个异常类继承了java.lang.Exception，他包含了C++中的Status类，用于描述任何Rocksdb的错误*

# 读写

数据库提供put,remove,和get方法用于修改、查询数据库。例如，下面的代码吧存储在key1的值移动到key2中

```java
byte[] key1;
byte[] key2;
// some initialization for key1 and key2

try {
  final byte[] value = db.get(key1);
  if (value != null) {  // value == null if key1 does not exist in db.
    db.put(key2, value);
  }
  db.remove(key1);
} catch (RocksDBException e) {
  // error handling
}

```

*TIP：调用RocksDB.put(WriteOptions opt, byte[] key, byte[] value) 和 RocksDB.get(ReadOptions opt, byte[] key)，你可以通过WriteOptions和ReadOptions控制put和get的行为*

*TIP：使用int RocksDB.get(byte[] key, byte[] value)或者int RocksDB.get(ReadOptions opt, byte[] key, byte[] value)，来避免在RocksDB.get()中创建一个byte数组，这两个函数的输出会填充到预分配好的输出缓冲区value中，返回的int表示value的实际长度。如果返回的长度大于value.length，意味着输出缓冲区的大小不够*

# 打开一个带有列族的数据库

一个rocksdb数据库可以有多个列族。列族允许你把类似的键值对放在一起，与其他列族独立进行操作。

如果你以前使用过Rocksdb但是没有显式使用过列族，你可能惊奇地发现，你的所有操作都发生在一个列族，这个列族名为“default”

在RocksJava中使用列族的时候，一个非常重要的注意点就是，在关闭数据库的时候，需要遵从一个非常特别的顺序来析构，保证资源的正确释放。这个顺序可以通过下列代码来说明：

```java
...
    // a static method that loads the RocksDB C++ library.
    RocksDB.loadLibrary();

    try (final ColumnFamilyOptions cfOpts = new ColumnFamilyOptions().optimizeUniversalStyleCompaction()) {

      // list of column family descriptors, first entry must always be default column family
      final List<ColumnFamilyDescriptor> cfDescriptors = Arrays.asList(
          new ColumnFamilyDescriptor(RocksDB.DEFAULT_COLUMN_FAMILY, cfOpts),
          new ColumnFamilyDescriptor("my-first-columnfamily".getBytes(), cfOpts)
      );

      // a list which will hold the handles for the column families once the db is opened
      final List<ColumnFamilyHandle> columnFamilyHandleList =
          new ArrayList<>();

      try (final DBOptions options = new DBOptions()
          .setCreateIfMissing(true)
          .setCreateMissingColumnFamilies(true);
           final RocksDB db = RocksDB.open(options,
               "path/to/do", cfDescriptors,
               columnFamilyHandleList)) {

        try {

          // do something

        } finally {

          // NOTE frees the column family handles before freeing the db
          for (final ColumnFamilyHandle columnFamilyHandle :
              columnFamilyHandleList) {
            columnFamilyHandle.close();
          }
        } // frees the db and the db options
      }
    } // frees the column family options
...
```

