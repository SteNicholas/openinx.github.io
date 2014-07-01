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



### 相关中间件汇总

Google 搜了一大圈，大概有这么些公司或者产品在做以上的相关事情: 

* youtube用GO开发的 [vitess](https://github.com/youtube/vitess.git)

* 淘宝的[cobar](https://github.com/alibaba/cobar.git) & [TDDL](https://github.com/alibaba/tb_tddl.git)

* 最近官方新出的 [mysql-fabric](http://dev.mysql.com/doc/mysql-utilities/1.4/en/fabric.html)

* Twitter的 [gizzard](https://github.com/twitter/gizzard)

* QiHoo360 的 [Atlas](https://github.com/Qihoo360/Atlas.git)

* [amoeba](http://sourceforge.net/projects/amoeba/files/Amoeba%20for%20mysql/)

* [Trove](https://wiki.openstack.org/Trove)是Openstack下的一个DAAS(Database As A Service)项目。目前提供的功能有： RestFullAPI操作高可用的数据库实例的创建，配置，备份，恢复，监控。 它依赖了Openstack里面的很多服务组件和代码包：

   * Keystone -- 权限认证服务
   * Nova     -- Iaas层支撑
   * Glance   -- 数据库Server的Image服务
   * Swift    -- 对象存储服务（存放Snapshot和Logs等大文件数据，用于数据备份和恢复）
   * Cinder   -- 块设备服务（远程挂载块设备形式完成数据备份和数据恢复)

*  [parelastic](http://www.parelastic.com/)
   基于OpenStack的DaaS产品[Trove](https://wiki.openstack.org/Trove)上做二次开发之类的。他们公司的产品就是一个类Trove的项目叫做[Tesora](https://github.com/Tesora/tesora-trove). 

*  [scalebase](http://www.scalebase.com/) 就是一个卖MySQL中间件的公司。 他们卖的商业版本中间件号称对客户端做到100%的透明，主要支持：

    * 数据库实例的scal-out
    * 事务的两阶段提交回滚
    * ACID
    * 跨分片JOIN, Order by, Group By, Limit和聚合。
    * auto-failover & auto-swithover

   按照[官方功能表](http://www.scalebase.com/products/benefits/)来看的话，基本上MySQL分布式中间件该有的问题，它都解决了。不知道是不是真的?

* [scalearc](http://www.scalearc.com/product/) 不是走的scalebase那套完美中间间方案。 更像是一个MYSQL集成管理工具。 它可以做： 

    * auto-failover
    * 查询缓存
    * 数据库层动态负载均衡。
    * 即时监控
    * SQL防火墙
    * 调用各大云服务厂商API，自动Scale-out.

