---
layout: post
title: "LevelDB-Notes"
description: ""
category: 
tags: [ leveldb ]
---

### LevelDB特点

1. 持久化
2. 自定义xcompare比较函数
3. 接口简洁， Write, Read, Delete, Batch.
4. read一致性snapshot.
5. 数据压缩 - snappy
6. 写性能40W/s, 读性能6w/s.
7. 日志-系统崩溃恢复
8. Restart Point - 前缀压缩
9. MemTable是用SkipList实现的。[SkipList的期望复杂度的证明](http://pan.baidu.com/s/1sjqGczF)
10. Cache: TableCache & BlockCache.
11. LevelDB在插入删除过程中，版本如何维护?
12. Minor Compaction & Major Compaction
13. bloom filter

### table/iterator.cc

注册多个清理function组织成链表, 在Iterator的回收时，依次执行每个清理函数.

### table/iterator_wrapper.h

IteratorWrapper缓存了iterator的it->Valid(),it->Key()两个状态。当要多次用到valid和key的值时，可以减少因调用it->Valid()和it->Key()而产生的开销。对外暴露的interface和Iterator一样。

### table/merger.h

将N个有序迭代器合并成一个有序的迭代器，找N个中最小的值时作者用的线性查找，可用MinHeap优化，降低时间复杂度。类似于这个问题[merge-k-sorted-lists](https://oj.leetcode.com/problems/merge-k-sorted-lists/)。

### LevelDB/SSTable编码图详细

![Alt TableFormat.png](/images/TableFormat.png)

### util/arena.cc

简单内存管理器，内存按照4K分配。一个个的4K块组成了整个的内存列表。提供内存对齐分配，对中间的空隙内存直接丢掉不管。


### util/bloom.cc


* 用于判断一个给定的key是否在一个键值集合set中。假设set = ['hello','world', 'good','boy'], 将set的每一个单词hash成一个[0,63]之间的数, 假设h(hello)=3,h(world)=24, h(good)=54, h(boy)=59。那么得到一个bit串,将这个串保存下来，取名叫做bloomFilter. 对给定一个key='caonima' , 按照同样的hash函数，得到一个[0,63]之间的数36,发现在bloomFiler的36位不为1，那么'caonima'肯定不在set这个集合中。为了更快的排除key不再set中，可以将set中的key进行t(1<=t<=30)次哈希取或，得到bloomFilter值。然后对key进行t次哈希取或，假设某一次哈希值对应的bit位不为1，就可以确定key不再set中。

```cpp
00010..010..010..010000
   ^    ^    ^    ^
   3    24   54   59
```

* h是个32为整数， `(h >> 17) | (h << 15)`将h的比特为向右Rotate17位。

### util/cache.cc

* LRU-Cache. 作者自己实现了一个可以resize的hash表，号称要比C++自带的哈希表快5%。个人觉的快就快在hash表桶中存放的节点LRUHandle，保存了key对应的hashValue。这样不论是在find操作还是resize操作都不要计算hash值，快了好多。
* LRUCache用双向链表+哈希表实现的话，get和set的复杂度都为O(1).
* ShardedLRUCache有4个普通的LRUCache组成，对key的hashValue取高4位来决定到底落在哪个LRUCache表里面。

### util/coding.cc

* 解决大端小端编码问题。

### util/env_posix.cc

* 实现了随机读写文件，顺序读写文件，mmap映射读写文件三种IO方式。
* 实现了生产者消费者模型的任务调度器。

### util/histogram.cc 

写了个Demo调用了下，发现生成了一个如下柱型图，用来做Benchmark的。

```
Count: 1000000  Average: 1.9963  StdDev: 1.42
Min: 0.0000  Median: 2.4955  Max: 4.0000
------------------------------------------------------
[       0,       1 )  201310  20.131%  20.131% ####
[       1,       2 )  199654  19.965%  40.096% ####
[       2,       3 )  199869  19.987%  60.083% ####
[       3,       4 )  199772  19.977%  80.061% ####
[       4,       5 )  199395  19.940% 100.000% ####
```

### util/random.h
	
* 简单的随机数生成器

