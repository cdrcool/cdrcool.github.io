---
title: 数据结构--LSM
date: 2019-10-27 20:26:33
categories: 算法与数据结构
---
## 概念
LMS，即 log-Structured Merge-Trees，其本质就是在读写之间取得平衡，和 B+ 树相比，它牺牲了部分读性能，用来大幅提高写性能。

## 原理
LSM 的原理是把一颗大树拆分成 N 棵小树，它首先写入到内存中（内存没有寻道速度的问题，随机写的性能得到大幅提升），在内存中构建一颗小树，随着小树越来越大，内存的小树会 flush 到磁盘上。当读时，由于不知道数据在哪颗小树上，因此必须遍历所有的小树，但在每颗小树内部数据是有序的。

## 优化
LSM Tree 弄了很多个小的有序结构，比如每 m 个数据，在内存里排序一次，下面 m 个数据，再排序一次......这样依次做下去，就可以获得 N/m 个有序的小的有序结构。
在查询的时候，因为不知道这个数据到底是在哪里，所以就从最新的一个小的有序结构里做二分查找，找得到就返回，找不到就继续找下一个小有序结构，一直到找到为止。
很容易可以看出，这样的模式，读取的时间复杂度是 (N/m) * log2N。读取效率是会下降的。
这就是最本来意义上的 LSM tree 的思路。那这样做，性能还是比较慢的，于是需要再做些事情来提升。

### Bloom filter
Bloom filter 是一种随机数据结构，可以在 O(1) 时间内判断一个给定的元素是否在集合中。False positive 是可能的，但是 false negative 是不可能的。
使用 Bloom filter，可以快速的知道某一个小的有序结构里有没有指定的那个数据的。于是就可以不用二分查找，而只需简单的计算几次就能知道数据是否在某个小集合里。效率得到了提升，但付出的是空间代价。

### Compaction
如果我们一直对 memtable 进行写入，memtable 就会一直增大知道超出服务器的内部限制。所以我们需要把 memtable 的内存数据放到 durable storage
上去，生成 SSTable 文件，这就做 minor compaction。

* Minor compaction
把 memtable 的内容写到一个 SSTable。目的是减少内存消耗，另外减少数据恢复时需要从日志读取的数据量。

* Merge compaction
把几个连续 level 的 SSTable 和 memtable 合并成一个 SSTable。目的是减少读操作要读取的 SSTable 数量。

* Major compaction
合并所有 level 上的 SSTable 的 merge compaction。目的在于彻底删除 tombstone 数据，并释放存储空间。

![LSM Compaction示例](/images/algorithm/LSM Compaction示例.jpg)

