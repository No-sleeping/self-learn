### 预备知识

1.一个扇区：512byte。系统读盘的时候，连续多个扇区一起读，也就是一个块。常见块大小是4KB=8个扇区。

2.inode：存储文件元信息。

3.元信息 → inode；数据 → block

### buffer cache & page cache

![](https://img-blog.csdnimg.cn/img_convert/65c7167ea2a2cbad0f453bfcff005117.png)

1.Page cache缓存文件的页以优化文件IO。 page cache → block。通常：4K

2.Buffer cache缓存块设备的块以优化块设备IO。buffer cache → inode。通常：1K

假设我们通过文件系统操作文件，那么文件将被缓存到Page Cache，如果需要刷新文件的时候，Page Cache将交给Buffer Cache去完成，因为Buffer Cache就是缓存磁盘块的。

简单说来，page cache用来缓存文件数据，buffer cache用来缓存磁盘数据。在有文件系统的情况下，对文件操作，那么数据会缓存到page cache，如果直接采用dd等工具对磁盘进行读写，那么数据会缓存到buffer cache。

***

linux 2.4.10之前 分两种 cache（buffer、page）。

2.4.10之后的buffer 放在page中统计了。**也就是说 2.4.10之后的内核的disk cache 只有 page cache, 而page cache中有些页面被叫做buffer page。**


***

释放缓存：

echo 1 > /proc/sys/vm/drop_caches : 释放page cache

echo 2 > /proc/sys/vm/drop_caches : 释放 inode cache和dentry cache

echo 3 > /proc/sys/vm/drop_caches : 释放全部
