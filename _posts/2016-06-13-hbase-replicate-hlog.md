---
layout: post
title: "HBase回放Hlog顺序不一致的问题"
description: ""
category: 
tags: [读书]
---


在HBase的主从复制集群中, 如下图左所示，Region-Server-X以及Region-Server-Y是master集群中的两个Region-Server。正常情况下， 对Region-A的写入会在Region-Server-X上append log 到Hlog-X，然后Region-Server-X会异步地将该部分Hlog批量地应用(apply)到slave-cluster中。 此时，若Region-Server-X发生了宕机，那么Region-A会被Region-Server-Y托管，之后业务开始写Region-A导致append log到Hlog-Y日志， 同时 Region-Server-Y会新开一个线程去执行Hlog-X的replay任务，这样就会出现Region-A的Hlog-X以及Hlog-Y同时写入slave-cluster的情况出现。也就是说, Region-A的Hlog在slave cluster中的回放顺序错乱。

另外，Region Move也会产生类似的问题，即两个Region Server并发回放Hlog导致回放顺序错乱。


<center>
<img src="/images/hlog-concurrent-replay-hlog.png" width="70%"> 
</center>

当Region-A的Hlog日志回放顺序错乱时，会导致主从集群最终数据不一致的问题：在master cluster上，先执行put操作，然后执行delete操作（delete操作是为了删除put的记录），在slave cluster中回放的顺序可能变成先执行delete 操作， 再执行put操作。如果在delete操作之后put操作之前，slave cluster的region-server 做了一次major compaction， 那么会导致出现put的数据没有被删除的情况。

另外，日志回放数据错乱还可能会导致slave-cluster数据处于一个从未在maser-cluster上出现过的状态。

该问题现在依然存在于hbase的系统中，社区也对此进行过多次讨论，但现在依然没有解决。下面简单阐述小米HBase团队提出的一种解决思路:

1. 把Region-A 从Region-Server-X上move 到Region-Server-Y，让Region-Server-Y托管Region-A。此时Region-A可读可写，但Region-A产生的Hlog-Y并不会立刻推送到slave-cluster上；
2. 某个Region-Server将Hlog-X中的Region-A的日志推送到slave-cluster；
3. 等待Hlog-X中Region-A的日志全部回放到slave-cluster之后，Region-Server-Y开始推送HLog-Y日志到slave-cluster上。


这里，还需要考虑一种比较极端的情况。如下图所示，有X , Y , Z 三个Region-Server, 其中X负责A/B/C 3个region, Y负责D/E 2 个region,  Z 负责F 1个region。若此时X发生crash， A/B 2个region被迁往Z，C这个region被迁往Y，之后Y开始接收Region-C的读写请求，在C写了一小段数据之后，Y这个Region-Server又发生了宕机，于是D/E/C 3个region全部都被迁移到Z。

<center>
<img src="/images/hbase-replay-log-seq.png" width="50%"> 
</center>


对于Region-C而言，为了保证Hlog推送到slave-cluster严格有序，必须先推送Hlog-X中的日志，然后再推送Hlog-Y中的日志，最后推送Hlog-Z的日志。对于Region-Server宕机的这种情况而言，由于宕机的Hlog并不会增长，因此依次回放Hlog-X/Hlog-Y中所有的日志即可；但对于Region迁移这种情况（仍以上图为例，假设Region-C从X迁移到Y），Region-Sever-X的Hlog-X会不断增长，因此在Region-Server-Y托管Region-C时需要记录下Region-C在Hlog-X中的MaxSequenceId，当回放的HLog-X的seqId>=MaxSequenceId时，就可以开始回放Hlog-Y的日志了。

因此，为了统一处理，无论region failover 还是region move，需要做以下记录：

1. Y从X上托管Region-C时，记录Hlog-X的MaxSequenceId到HBase的meta表中；
2. 在Z从Y上托管Region-C时，记录Hlog-Y的MaxSequenceId到HBase的meta表中。
3. ......


极端情况下，一个Region在多个Region-Server之间做多次迁移，会形成一条MaxSequenceId链表，为了保证改Region的Hlog回放顺序严格一致，必须依序回放各个Region-Server对应的Hlog，直到该段Hlog回放到对应的MaxSequenceId，接着回放下一段hlog。这样就能保证HLog在Region Move/Region Server Failover时，Region的回放顺序严格一致了。


目前该方案的设计方案已经发到社区了，小米HBASE团队也将对此进行修复。对该问题的更多相关讨论可以参考[HBASE-9465](https://issues.apache.org/jira/browse/HBASE-9465)。


### 总结
本文介绍了一种保证Hlog回放顺序严格一致的方案，可以解决master-cluster和slave-cluster数据不一致的问题。该方案可能会导致region在迁移过程中master-cluster和slave-cluster复制延迟增大（新方案必须严格等待上一段hlog回放，才能回放下一段hlog），整个过程相当于牺牲了主备集群replication的及时性，换来主从集群间数据最终一致性。

### 参考资料

1. https://issues.apache.org/jira/browse/HBASE-9465
2. https://issues.apache.org/jira/browse/ACCUMULO-2931
3. https://issues.apache.org/jira/browse/HBASE-10275
4. https://hbase.apache.org/0.94/replication.html







