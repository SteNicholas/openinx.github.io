---
layout: post
title: "MySQL相关总结"
description: ""
category: 
tags: [ mysql ]
---

### MySQL相关总结

算起来，做MYSQL相关的工作已经做了一年半了。平时上班的时候, 虽然运维，开发，DBA一起搞，但是哪一方面都没有干好。 即便如此还是写点东西，原因如下： 

* 有时候跟朋友扯淡， 说数据库这方面有一部分依赖经验。 
* 不少用MYSQL的同学，对MySQL确实缺乏一些基本的理解。 虽然以在下的水平去忽悠专业搞MySQL内核水平的人，显然是不太现实，但是忽悠下开发感觉还是不错。 
* 虽然被朋友鄙视说 “天天改几个MySQL参数调优或者做运维监控备份什么的 ，简直闲得蛋疼菊紧！“，但是不可忽视的现象是， MySQL在被大量的企业应用，早在2007年就有25%的市场占用率。

所以总结下，免的今后又重新来过。

### 监控工具

* glances

  绝对是linux下的任务监控利器。比windows下的*任务管理器*强大数倍。直接一个glances命令可以看到： 

  * 动态显示每个Process的CPU, Memory, IO-read, IO-write的数据量
  * 每个网卡每秒读写数据
  * 每个磁盘设备的每秒读入读出

* top

* dstat  大家笑称 *屌丝状态*

* iostat -x 1

  这个查看IO必不可少。该命令显示的每一列，最重要的要看%util，await两列。当%util超过80%,基本就IO被打满了。await大于1基本说明IO写入有点慢了。 

* zabbix

  有一个agent不断收集本地服务器上的系统数据和mysql相关数据，发送到远端的中央处理服务。 它可以按照时间轴展示各个指标的曲线图。 我们这边基本都是服务器标配了。

### 关键参数

* innodb_buffer_pool_size=机器内存的75%

* binlog_format=ROW 

* innodb_file_per_table=TRUE

* innodb_log_file_size=1.8G  & innodb_log_files_in_group=2

  总的 redo log file size = innodb_log_files_in_group * innodb_log_file_size . 就性能而言，redo log file size能尽量大就尽量大。因为mysql5.5主要为32位机器设计， 所以最大为4GB（why?）, 在mysql5.6中最大可以设置为512GB。 

* innodb_write_io_threads = CPU-cores * 2

* innodb_read_io_threads= CPU-Cores*1/3 

* innodb_thread_concurrency= CPU-cores * 2/3 

* innodb_flush_log_at_trx_commit = 1 & sync_binlog = 1

  对事务要求严格的业务场景，这两个参数都必须设置为1. 详细说明： 

  * [innodb_flush_log_at_trx_commit](http://dev.mysql.com/doc/refman/4.1/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)
  * [sync_binlog ](http://dev.mysql.com/doc/refman/5.0/en/replication-options-binary-log.html#sysvar_sync_binlog )

* innodb_flush_method=O_DIRECT

### 性能调优

* mysql> show engine innodb status\G

  该命令显示了非常详细的innodb的运行状态信息。 最值得开发人员关注的就是这一段:

        ------------
        TRANSACTIONS
        ------------
        Trx id counter AFB
        Purge done for trx's n:o < 423 undo n:o < 0
        History list length 61
        LIST OF TRANSACTIONS FOR EACH SESSION:
        ---TRANSACTION AF7, ACTIVE 1 sec
        1 lock struct(s), heap size 376, 0 row lock(s), undo log entries 65
        MySQL thread id 310, OS thread handle 0x7f8cda319700, query id 98117 
            localhost 127.0.0.1 root
        TABLE LOCK table `test`.`fake_data` trx id AF7 lock mode IX


  这一段显示了当前每个事务的一下信息： 

  * 截至到目前位置的运行时间： ACTIVE 1 sec。假设一个事务运行时间太长，又尚未结束。那么看这个数值，肯定可以看出来。 
  * 事务的undo log条数: undo log entries 65 . 举个简单的例子， 假设一事务要插入100条记录，现在插入了82条，那么undo log 就为82. 
  * 该事务用了哪些锁： X , S, IX , IS, Next-Key lock, Gap Lock, Record Lock 等等 
 

* mysql> explain *Your-Select-SQL*


        mysql> explain select * from fake_data\G
        *************************** 1. row ***************************
                   id: 1
          select_type: SIMPLE
                table: fake_data
                 type: ALL
        possible_keys: NULL
                  key: NULL
              key_len: NULL
                  ref: NULL
                 rows: 9649329
                Extra: 
        1 row in set (0.00 sec)
                mysql> explain select * from fake_data ;  


   这个也是非常值得开发人员学习的一个命令，可以看到select SQL的如下信息： 

   * possible_keys : SQL有哪些索引可以选择
   * key: 最终做代价评估之后，优化器选择走的索引
   * rows: 该SQL将扫描表的行数


* mysql> show index from *YouTableName*


        mysql> show index from fake_data\G
        *************************** 1. row ***************************
                Table: fake_data
           Non_unique: 0
             Key_name: PRIMARY 
         Seq_in_index: 1
          Column_name: id
            Collation: A
          Cardinality: 9649202
             Sub_part: NULL
               Packed: NULL
                 Null: 
           Index_type: BTREE
              Comment: 
        Index_comment: 
        1 row in set (0.00 sec)


   这个太常用了， 查看表的索引相关的信息。
   尤其值得关注的一列是 Cardinality 。 这列表明了该字段的在全表数据中的 *散列性* 。 
   所谓 *散列性* 就是你一个全表数据，讲该列字段单独取出来，组成一个集合，去掉重复值之后，集合中值总数。
   字段唯一的情况下，这个列应该约等于改表的总行数。 散列性越高说明，这个字段上建索引越高效。


* orzdba 
   淘宝90后dba陈旭开发mysql工具。号称mysql性能医生。在各种工具中，确实这个是最好用的。常用的命令有: 

   * orzdba -my 
   * orzdba -innodb

* innotop 

   该工具只是对`show engine innodb status`这个命令的数据做了一个总结，做的就像top这个命令一样。 

* dd 

   当发现写入性质的SQL( insert , update, delete等) 都非常慢时，第一反应就是用这个命令测试下磁盘的读写速度。 

### 备份

* mysqldump
* innodbackupex
* lvm备份

### 复制

较长的宕机时间和丢失数据 中选择一个。 

* 异步
* 同步
* 半同步 

  半同步主要有两个特点： 

  * 多个slave和一个Master同步时， 只有有一个slave向Master返回写入relay-log的Ack，Master便认为这个事务成功了。 
  * 当超过一定时间之后，Master依然没有收到其中任何一个Slave的ack, 那么这时候会转化成异步复制

* 虚拟同步

![Alt sync.png](/images/sync.png)

### MYSQL好点的书籍
_话说其他关于MYSQL的书，基本都渣的一塌糊涂_

* [高性能MYSQL](http://book.douban.com/subject/4241826/)
* [高可用MYSQL](http://book.douban.com/subject/6847455/)
* [MySQL技术内幕： Innodb引擎](http://book.douban.com/subject/24708143/) 
* [数据库系统实现](http://book.douban.com/subject/4838430/) 
* [数据库查询优化器的艺术](http://book.douban.com/subject/25815707/)
* [事务处理](http://book.douban.com/subject/3651015/)

姜承尧 & 李海翔 两位大神的书，我基本都愿意看，包括这本 [MySQL内核：InnoDB存储引擎 卷1](http://book.douban.com/subject/25872763/) 我第一时间（2014-5.22发售）赶紧买来。 

