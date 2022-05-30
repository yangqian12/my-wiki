### HDFS 中的 hflush, hfsync

---

hadoop 2.0 之前的版本

**sync | hflush**

将 client 端写入的数据刷到每个 datanode 的 OS 缓存中。返回之后，可以看到最新的数据，但是这个方法不保证数据持久化，只保证数据可见性。

hadoop 2.0 提供了 (sync, hflush, hsync) 三种方法

**hsync 语义**

将 client 端写入的数据刷到每个 datanode 的磁盘中（hsync）。比较耗时。