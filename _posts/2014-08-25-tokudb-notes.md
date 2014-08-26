---
layout: post
title: "TokuDB-Notes"
description: ""
category:
tags: [ tokudb ]
---

### TokuDB主要特点
1. 分形树索引
2. ACID & MVCC
3. Multiple Clustering Indexes 
4. SSD上写性能甩Innodb一截
5. Hot Schema Changes. 加索引的同时，修改数据。
6. 相比InnoDB更少的数据存储空间
7. 同步复制的延时更少
8. 每个节点都有buffer
