---
layout: post
title: "MySQL中间件关注的一些问题"
description: ""
category:
tags: [mysql, mysql-cluster]
---

### Sharding

* 分片是全自动？半自动？必须手动？ 
* 在客户端实现分片路由，还是在中间件实现分片路由？ 
* 大表记录的分片规则是？Hash ?  Range ? 权重路由？ 一致性hash? 
* Shard-Split? 
* Shard-Move？
* 分片路由信息是否对客户端透明？ 假设不透明，是如何通知客户端更新路由？ 
* 分片挂掉，如何保证高可用的？ 
* 是否支持cross-shard的SQL操作?
* 是否支持cross-shard的事务？ 一个事务中有几条SQL在A分片，另几条SQL在B分片，能否处理？

### HA(High-Availbilty)

* 主从节点间使用的哪种复制方式？同步？ 异步？ 半同步？ 虚拟同步？ 
* HA拓扑结构？ M-S-S ？ M-M ？ 
  * 基于拓扑结构,如何实现读写分离? 
  * 一个事务中一条SQL为read, 另一条SQL为Write,如何处理？
* 如何检测到Master的failure? 检测到failure的时间长度？ 
* 检测到failover之后，怎么通知协调其他的Slave节点?
* 在多个Slave节点中，选择Slave节点提升为Master的策略是什么？
* Slave节点提升为Master节点过程中，是否会丢失数据？ 
* 切换过程中是否对客户端透明？ 假设不透明，服务中断时长和哪些因素有关？ 

