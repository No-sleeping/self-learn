预备知识：

一个扇区：512byte。

系统读盘的时候，连续多个扇区一起读，也就是一个块。常见块大小是4KB=8个扇区。

inode：存储文件元信息。

https://blog.csdn.net/xuz0917/article/details/79473562

查看inode信息



元信息 → inode
数据 → block
Page cache缓存文件的页以优化文件IO。 page cache → block

Buffer cache缓存块设备的块以优化块设备IO。buffer cache → inode



page cache：缓存文件数据。通常：4K

buffer cache：缓存磁盘数据。通常：1K



假设我们通过文件系统操作文件，那么文件将被缓存到Page Cache，如果需要刷新文件的时候，Page Cache将交给Buffer Cache去完成，因为Buffer Cache就是缓存磁盘块的。

简单说来，page cache用来缓存文件数据，buffer cache用来缓存磁盘数据。在有文件系统的情况下，对文件操作，那么数据会缓存到page cache，如果直接采用dd等工具对磁盘进行读写，那么数据会缓存到buffer cache。



参考：https://www.baidu.com/s?ie=UTF-8&wd=buffer%20cache/%20page%20cache

linux 2.4.10之前 分两种 cache（buffer、page）。

2.4.10之后的buffer 放在page中统计了。也就是说 2.4.10之后的内核的disk cache 只有 page cache, 而page cache中有些页面被叫做buffer page。

释放缓存：

echo 1 > /proc/sys/vm/drop_caches : 释放page cache
echo 2 > /proc/sys/vm/drop_caches : 释放 inode cache和dentry cache
echo 3 > /proc/sys/vm/drop_caches : 释放全部



![buffer](https://github.com/No-sleeping/self-learn/blob/main/images/Kernel/buffer.jpg)
