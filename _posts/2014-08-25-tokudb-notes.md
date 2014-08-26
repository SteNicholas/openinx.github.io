---
layout: post
title: "TokuDB-Notes"
description: ""
category:
tags: [ tokudb ]
---

### TokuDB主要特点
1. fractal tree storage
2. Transactions with MVCC
3. Multiple Clustering Indexes 
4. SSD上写性能甩Innodb一截
5. DDL操作很快， 比如容许在加索引的同时，修改数据。
6. 相比InnoDB更少的数据存储空间
7. 同步复制的延时更少

