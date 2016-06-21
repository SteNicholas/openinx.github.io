---
layout: post
title: "HBase Region Balance实践"
description: ""
category: 
tags: [ hbase ]
---


在公司的HBase集群中，HBase节点和HDFS的节点基本上都是混合部署的，因为HDFS是磁盘密集型业务，HBASE的HMaster以及RegionServer都是CPU密集型业务，混合部署可以到达CPU和磁盘空间得到充分利用的目的。最近在在维护公司的集群的时候，碰到一个需求就是对hbase的集群进行扩容，本次计划是打算扩容4台800G * 12 的SSD机器， 那么在扩容过程中需要注意什么呢？ 


### balance的流程

* 首先找出所有需要move的plan . 寻找plan的规则.
* unassign region . 
* assign region . 


unassign region的具体流程为：


* create closing node . 
* hmaster 调用rpc服务关闭region server。region-close的流程大致为先获取region的writeLock ， 然后flush memstore, 再并发关闭该region下的所有的store file文件(注意一个region有多个store，每个store又有多个store file , 所以可以实现并发close store file) 。最后释放region的writeLock.
* 


assgin region的具体流程为: 

* 设置table为enable状态 
* HMaster通过调用rpc服务open region。region-open的流程大致为在zk上记录regioin状态为opening状态。
* 更新region 状态为 OPEN. 


### balance需要注意的坑

第一个需要注意的问题问题就是数据本地化的问题。 
第二个问题就是需要适当设置好balance操作的超时候时间。 


### 总结



### 参考资料: 


