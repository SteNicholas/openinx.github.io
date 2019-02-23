---
layout: post
title: "谈谈Netty的内存管理"
description: ""
category: 
tags: []
---

最近在重新梳理HBase的读写全路径Offheap的思路。在我的性能测试结果中发现，100% Get的场景受Young GC的影响仍然比较严重，在[HBASE-21879](https://issues.apache.org/jira/browse/HBASE-21879)贴的两幅图中，可以非常明显的观察到Get操作的p999延迟跟G1 Young GC的耗时基本相同，都在100ms左右。按理说，在[HBASE-11425](https://issues.apache.org/jira/browse/HBASE-11425)之后，应该是所有的内存分配都是在offheap的，heap内应该几乎没有内存申请。但是，在仔细梳理代码后，返现从HFile中读Block的过程仍然是先拷贝到堆内去的，一直到BucketCache的WriterThread异步地把Block刷新到Offheap上堆内的DataBlock才释放。而磁盘型压测试验中，由于数据量大，Cache命中率并不高(~70%)，所有会有大量的Block读取走磁盘IO，于是Heap内产生大量的年轻代对象，最终导致Young区GC压力上升。

消除Young GC直接的思路就是从HFile读DataBlock的时候，直接往Offheap上读。之前留下这个坑，主要是HDFS不支持ByteBuffer的Pread接口，当然后面开了[HDFS-3246](https://issues.apache.org/jira/browse/HDFS-3246)在跟进这个事情。但后面发现的一个问题就是：Rpc路径上读出来的DataBlock，进了BucketCache之后其实是先放到一个叫做RamCache的临时Map中，而且Block一旦进了这个Map就可以被其他的RPC给命中，所以当前RPC退出后并不能直接就把之前读出来的DataBlock给释放了，必须考虑RamCache是否也释放了。于是，就需要一种机制来跟踪一块内存是否同时不再被所有RPC路径和RamCache引用，只有都不引用的情况下，才能释放内存。自然而言的想到用reference Count机制来跟踪ByteBuffer，但发现其实Netty依赖已经较完整地实现了这个东西，于是看了一下Netty的内存管理机制。

Netty作为一个高性能的基础框架，为了保证GC对性能的影响降到最低，做了大量的offheap化。而offheap的内存是程序员自己申请和释放，忘记释放或者提前释放都会造成内存泄露问题，所以一个好的内存管理器很重要。首先，什么样的内存分配器，才算一个是一个“好”的内存分配器：

1. 线程安全。一般一个进程共享一个全局的内存分配器，得保证多线程并发申请释放既高效又不出问题。
2. 高效的申请和释放内存，这个不用多说。
3. 方便跟踪分配出去内存的生命周期和定位内存泄露问题。
4. 高效的内存利用率。有些内存分配器分配到一定程度，虽然还空闲大量内存碎片，但却再也没法分出一个稍微大一点的内存来。所以需要通过更精细化的管理，实现更高的内存利用率。
5. 尽量保证同一个对象在物理内存上存储的连续性。例如分配器当前已经无法分配出一块完整连续的70MB内存来，有些分配器可能会通过多个内存碎片拼接出一块70MB的内存，但其实合适的算法设计，可以保证更高的连续性，从而实现更高的内存访问效率。

<img src="/images/netty-chuck-allocation.png" width="100%">


参考资料

1. https://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf
2. https://www.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/%0A%20%20%20480222803919
3. https://netty.io/wiki/reference-counted-objects.html