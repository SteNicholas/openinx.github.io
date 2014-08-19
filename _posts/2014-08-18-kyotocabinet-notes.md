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
2. serializable和read uncommitted两种隔离级别. 弱爆了。。。
3. hash database: record lock
4. B+ tree: page lock; SequeitialAccess优于RandomAccess
5. Prefix search ; regex search ; logging ; hot backup ; pseudo-snapshot ; merging
6. pseudo-snapshot会阻塞写线程.
7. 1个接口，7个实现


### kchashdb.h 

![Alt tchhashdb.png](/images/tchhashdb.png)
