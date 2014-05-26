---
layout: post
title: "About Huker"
description: ""
category: 
tags: [ huker, mysql]
---


取名叫做Huker的原因:

* 首先是因为这个名字听上去还是挺酷的，Hack们都喜欢酷酷的名字的项目，不是吗？ 

* 其次，在英文上讲，huker正好和"胡克"谐音， 假设有一个项目能和一个大科学家扯上点关系，顿时都会有种高大上的感觉。

* 最后，耍酷的孩子们都喜欢什么“客”之类的，比如: 

   * Hack们就喜欢Hacker(黑客),
   * 搞acm的都喜欢自称ACMer，
   * 网上爱好HTML收集截取的都叫做剪客，
   * 当然武侠中牛逼的人都喜欢自称剑客；
   * 在警卫员要收走周星驰腰间别着的一把杀猪刀时， 周赶紧说“对不起，身为一个刀客，刀不离身“，

  由此可见，Huker也跟客扯上点关系， 也能拉升点身价吧。 

### Huker到底是个啥玩意 

其实Huker是开源的RDS(Rational Database Service)作品。 有点类似与Openstack的trove项目，网易的RDS可能也和这个差不多。 但是没有trove那样太过于依赖Openstack的其他各个组件，比如cinder, swift等等。 
Huker目前只提供的MySQL关系型数据库的RDS服务。 未来可能会支持Mariadb, Postgresql等其他的开源数据库。 

它的主要的功能包括： 

* MySQL的定时自动备份，备份采用流式压缩上传到云存储服务。 (backup)

* 基于数据库的主从自动切换 (auto-failover)

* 数据库的灾难恢复，有两种方式： 

    * 基于指定镜像的恢复  (retore base on snapshot)
    * 指定实例，恢复到指定的历史时间点 （restore in point time）

* 动态的批量修改一批MySQL-Server的参数。 （parameter group）

* 提供数据服务器网络访问权限隔离（security group）

* 数据库性能和运行状态的实时监控报警 (system monitor)

### Huker的组成部分

* huker-api

  该服务主要是提供外界restful-api的访问， 关于api的详细文档可以参考[doc](https://github.com/openinx/huker/tree/master/doc). 其中大部分的api设计都参考了Openstack的api设计。

* huker-nodekeeper

  管理数据库实例的中央管理节点。 

* huker-sysmgr 

  在每个数据库实例所在的Linux服务器上，都部署了一个huker-sysmgr服务。主要用来做系统级别的管理操作，以及收集系统性能指标，监控和报警。 

* huker-guestagent
  每个MYSQL数据库服务，都有一个huker-guetagent来管理。 主要用来负责：

  * 主从关系的配置，主从心跳的维持
  * MySQL相关监控
  * 定时制作镜像
  * 定时上传日志文件

* huker-watcher 

  监控信息和报警信息的集中收集服务。基于HTTP支持较大并发和吞吐，设计简单，响应快速，便于拓展。

### Huker依赖的外部服务

* RabbitMQ 

  提供异步持久的消息队列服务。

  主要消息流向： 

  *  huker-api ====> huker-nodekeeper
  *  huker-api ====> huker-sysmgr 
  *  huker-api ====> huker->guestagent
  *  huker-nodekeeper ====> huker->sysmgr 
  *  huker-nodekeeper ====> huker->guestagent

 
* 大对象存储服务， 比如OpenStack的Swift, 或者其他的云存储服务。 
 
  提供基础数据存储服务, 存放的数据文件有：

  * 数据库数据镜像
  * 数据库增量日志文件
  * MySQL等数据库的二进制安装包
  * huker项目的python源码安装包及配置文件。

* Zookeeper

  主要用来负责MySQL服务的心跳管理和故障检测, zookeeper的客户端有： 

  * huker-nodekeeper 
  * huker-guestagent

### Huker TODO LIST

* 基于Docker轻量级的虚拟化容器实现方案
* 支持任何情况下，无数据丢失的主从切换。 

