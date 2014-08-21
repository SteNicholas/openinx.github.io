---
layout: post
title: "Kyotocabinet-Notes"
description: ""
category: 
tags: [ kyotocabinet ]
---

### Kyoto Cabinet特点

0. 多线程安全
1. 事务ACID 
2. serializable和read uncommitted两种隔离级别. 弱爆了
3. hash database: record lock
4. B+ tree: page lock; SequeitialAccess优于RandomAccess
5. Prefix search ; regex search ; logging ; hot backup ; pseudo-snapshot ; merging
6. pseudo-snapshot会阻塞写线程.
7. 1个接口，7个实现


### kchashdb.h 

`HashDB`特点:  
1. 由于HashDB桶的个数是静态的，后期没法拓展。所有读写性能直接和预设桶个数有关。  
2. 二次hash的哈希函数的散列程度。hash值随机性越好，桶内的二叉搜索树的高度越低（近似平衡二叉树）。读写的seek次数就越少。  
3. Record的碎片整理算法有关。碎片整理是为了让一个bucket里面的record查找，变成顺序读。所以碎片整理不能开销太大，成为IO短板。  
4. 打开文件的mmap的内存大小有关(在代码中是`msiz_`这个变量)。hashdb的IO封装有两个:`write_fast`和`read_fast`. 二者都是看读取的数据段的off和size这一段buffer，假设完全在mmap映射内存段内，直接读该内存，有超出内存映射段外的直接用pread和pwrite这两个系统调用。  
5. 没有用任何LRUCache.  

![Alt tchhashdb.png](/images/tchhashdb.png)

### kchplantdb.h

* B+ Tree的数据存储结构

![Alt KyotoCabinet-BPlusTree-dataFormat.png](/images/KyotoCabinet-BPlusTree-dataFormat.png)

* PlantDB的类组织结构

![Alt KyotoCabinetBPlusTreeClass.png](/images/KyotoCabinetBPlusTreeClass.png)

