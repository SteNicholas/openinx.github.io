---
layout: post
title: "Vitess解析"
description: "Vitess原理解析"
category: 
tags: [ golang, vitess, mysql, reshard, distribute database, database]
---


在线版本： [PPT](/ppt/vitess-notes.html)  
这是我在公司内部做的一次有关Vitess的技术分享, 要点有： 

* vitess提供的功能模块／特性／系统架构
* vitess的sharding方式
* vitess支持的SQL语法集
* vitess resharding的实现原理
* vitess 数据备份原理
* vitess 对比传统关系型数据库及NoSQL的优点和缺点



> 小插曲

为了尝试使用markdown制作在线的PPT， 我尝试了一些方案。最终选在了[remarkjs](http://remarkjs.com/)来制作在线版本的ppt，原因是:  

* 可以实现ppt文件的版本控制。这样在git中可以清晰的看到我的修改的增量。  
* remarkjs简单，只需要一个文件。 我尝试其他工具时，比如landslide， 会生成一堆文件，让我感到繁琐。  
* 可以使用markdown语法编辑。