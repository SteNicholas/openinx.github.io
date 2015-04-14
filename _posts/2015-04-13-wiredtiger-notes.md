---
layout: post
title: "WiredTiger Notes"
description: "WiredTiger Notes"
category: 
tags: [wiredtiger]
---

### WiredTiger
wired意为精力充沛的， tiger意为老虎。 合意为精力充沛的老虎。 

### WiredTiger特点
1. row存储， colume存储， LSM树
2. ACI事务， 标准隔离级别
3. hazard pointers, lock-free algorithms, fast latching, message passing
4. column-store formats, 前缀压缩和静态编码。 
5. 文档级锁 beat 集合级锁
6. 100%兼容MMAP, 可无缝迁移到WiredTiger.
