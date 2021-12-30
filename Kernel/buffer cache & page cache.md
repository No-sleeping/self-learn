# 预备知识：
一个扇区：512byte。
系统读盘的时候，连续多个扇区一起读，也就是一个块。常见块大小是4KB=8个扇区。
inode：存储文件元信息。
参考：https://blog.csdn.net/xuz0917/article/details/7947356

# Page cache缓存文件的页以优化文件IO。
# Buffer cache缓存块设备的块以优化块设备IO。

- page cache → block   缓存文件数据。通常：4K
- buffer cache → inode  缓存磁盘数据。通常：1K

![buffer](https://github.com/No-sleeping/self-learn/blob/main/images/Kernel/buffer.jpg)
