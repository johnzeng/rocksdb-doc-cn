# 什么时候需要他

当DB使用成吨的GB级别的内存的时候，很大可能当程序需要访问内存数据的时候，程序会遇到TLB未命中的情况和从映射取数据的时候缓存未命中的情况。当使用memtable和表读取器提供的基于哈希表的索引和bloom过滤器的时候，用户的感觉会更明显，因为数据的局部性非常糟糕。这些索引以及bloom都非常适合放在巨型页TLB中。当你看到TLB数据大量溢出，并且有巨型页功能支持的时候，考虑打开这个功能吧。

现在只在Linux支持这个功能。

# 如何使用

## 需求单
 
 - 你需要在linux中预留出巨型页
 - 知道可以使用的巨型页的大小

参考Linux的Documentation/vm/hugetlbpage.txt获得更多细节
 
## 配置

这里介绍这个功能在哪里，如何打开：

 - memtable的bloom过滤器：设置Options.memtable_prefix_bloom_huge_page_tlb_size为巨型页的大小
 - 哈希链表的memtable索引以及bloom过滤器：当调用NewHashLinkListRepFactory来创建一个memtable工厂对象的时候，把巨型页的大小通过huge_page_tlb_size传入
 - PlainTableReader的索引及bloom过滤器。当调用NewPlainTableFactory或者NewTotalOrderPlainTableFactory创建表工厂对象的时候，通过huge_page_tlb_size传入巨型页的大小

TLB: Translation lookaside buffer，地址转译缓冲。用于加快逻辑地址与物理地址转译速度的缓冲区。有硬件实现和软件实现两种。

