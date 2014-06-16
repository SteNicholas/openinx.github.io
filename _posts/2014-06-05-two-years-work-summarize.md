---
layout: post
title: "两年工作总结"
description: ""
category: 
tags: [ redis, zookeeper, leveldb, openstack, mysql ]
---

_鄙人一个毕业生，在公司混了两年，活脱脱混成一个傻逼。最经典的傻逼画面，莫过于被问到一个问题时，一双呆滞的眼睛加上一脸茫然木讷的表情..._

准备花几周时间总结下两年来工作上和业余学到的一些东西。 

* [redis](https://github.com/antirez/redis)

  2012年下半年，那时候就在那傻不垃圾的做 `分布式redis缓存云`， 找个机会在深入细致的剖析下redis的原理。 最好就画一张结构图， 把redis里面各个组件原理讲的清清楚楚。

* [kazoo](https://github.com/python-zk/kazoo) & [zookeeper](https://github.com/apache/zookeeper)


  * zookeeper各种奇淫巧技的用法。
  * 如何编写一个真正异步触发的zookeeper客户端
  * zookeeper的大致实现原理， 当年做 `分布式redis缓存云` 的时候，天天想着研究这个东西

* [leveldb](https://github.com/basho/leveldb)


  在朋友的推荐下，开始玩的leveldb , 后面看了一两遍代码， 虽然知道个大概， 但是总是没有达到理解起来一气呵成的境界。所以打算花点时间再看看。 


* [Openstack](https://github.com/openstack)


  我从2012年12月份的样子开始玩Openstack, 但一直局限在nova, keystone, glance, trove这四个组件。记的我们刚开始玩的时候，都还是F版本， 现在都到H版本了。 再深入玩玩H版本的其他各个组件。


* [Docker](https://github.com/dotcloud/docker)

  玩过，听过分享。

* [saltstack](https://github.com/saltstack/salt)


  配置管理和远程执行。 其实就是个自动化运维工具。工作上用了一段时间salt， 还是想真正找点时间能彻底理解下，这个东西是怎么做的。 


* mysql相关的东东： 

  * youtube用GO开发的 [vitess](https://github.com/youtube/vitess.git)
  * 淘宝的[cobar](https://github.com/alibaba/cobar.git) & [TDDL](https://github.com/alibaba/tb_tddl.git)
  * 最近官方新出的 [mysql-fabric](http://dev.mysql.com/doc/mysql-utilities/1.4/en/fabric.html)
  * Twitter的 [gizzard](https://github.com/twitter/gizzard)
  * QiHoo360 的 [Atlas](https://github.com/Qihoo360/Atlas.git)
  * [amoeba](http://sourceforge.net/projects/amoeba/files/Amoeba%20for%20mysql/)

* 参加过一个《数据库引擎开发》的课程，当时视频看了2/3, 好像还没看完。


  好歹弄个 B+ Tree实现的存储引擎的 K-V型数据库出来。 估计到时侯会是MongoDB的一个精简版本。 


* 收集了一些[德哥的Postgresql](http://blog.163.com/digoal@126/blog/static/16387704020141229159715/)一系列视频, 打算跟着德哥的视频混一段时间。 

