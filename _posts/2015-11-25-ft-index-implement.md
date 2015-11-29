---
layout: post
title: "TokuDB的索引结构--分型树的实现"
description: "TokuDB的索引结构--分型树的实现"
category: 
tags: [database, tokudb, mysql, storage engine]
---

> 本文从工程实现角度解析TokuDB的索引结构－－分型树。 详细描述了ft-index的磁盘存储结构，ft-index如何实现Point-Query, Range-Query,  Insert/Delete/Update操作,  并在描述过程中，试图从各个角度和InnoDB的B+树做详细对比。  

### 分型树简介

分型树是一种用来组织磁盘索引的树形数据结构，它是一种写优化的数据结构。 也就是说，在一般情况下， 分型树的写操作（Insert/Update/Delete）性能比较好(Percona公司测试结果显示, TokuDB的写性能优于InnoDB的[B+树](https://en.wikipedia.org/wiki/B%2B_tree))，同时它还能保证读操作近似于B+树的读性能（读性能略低于B+树）。 类似的索引结构还有LSM-Tree, 但是LSM-Tree的写性能远优于读性能。

工业界实现分型树最重要的产品就是[Tokutek](https://github.com/Tokutek)公司开发的ft-index（Fractal Tree Index）键值对存储引擎。这个项目自2007年开始研发，一直到2013年开源，代码目前托管在[Github](https://github.com/percona/PerconaFT)上。开源协议采用 GNU General Public License授权。 TokuMX公司为了充分发挥ft-index存储引擎的威力，基于K-V存储引擎之上，实现了MySQL存储引擎插件提供所有API接口，用来作为MySQL的存储引擎， 这个项目称之为[TokuDB](https://github.com/percona/tokudb-engine)， 同时还实现了MongDB存储引擎的API接口，这个项目称之为[TokuMX](https://github.com/Tokutek/mongo)。在2015年4月14日， Percona公司宣布收购Tokutek公司， ft-index/TokuDB/TokuMX这一系列产品被纳入Percona公司的麾下。自此， Percona公司宣称自己成为第一家同时提供MySQL和MongoDB软件及解决方法的技术厂商。

本文主要讨论的是TokuDB的 ft-index。 ft-index相比B+树的几个重要特点主要有：      

* 从理论复杂度和测试性能两个角度上看， ft-index的Insert/Delete/Update操作性能优于B+树。  但是读操作性能低于B+树。      
* ft-index采用更大的索引页和数据页（默认为4M, InnoDB默认为16K）， ft-index的数据页和索引页的压缩比更高。也就是说，在打开索引页和数据页压缩的情况下，插入等量的数据， ft-index占用的存储空间更少。  
* ft-index支持在线修改DDL (Hot Schema Change)。 简单来讲，就是在做DDL操作的同时(例如添加索引)，用户依然可以执行写入操作， 这个特点是ft-index树形结构天然支持的。 由于篇幅限制，本文并不对Hot Schema Change的实现做相关描述。    

此外， ft-index支持事务(ACID)以及事务的MVCC(Multiple Version Cocurrency Control 多版本并发控制)， 支持崩溃恢复。

正因为上述特点，  Percona公司宣称TokuDB一方面带给客户极大的性能提升， 另一方面还降低了客户的存储使用成本。
 
 
### ft-index的磁盘存储结构 
 
ft-index的索引结构图如下(在这里为了方便描述和理解，我对ft-index的二进制存储做了一定程度简化和抽象， 具体的二进制存储格式可以参考我的博客)：

<img src="/images/tokudb/ft-index-tree-structure.png" width="100%"> 

分型树的树形结构非常类似于B+树, 它的树形结构由若干个节点组成（我们称之为Node或者Block，在InnoDB中，我们称之为Page或者页）。 每个节点由一组有序的键值组成。假设一个节点的键值序列为[3, 8],  那么这个键值将(-00, +00)整个区间划分为(-00, 3), [3, 8), [8, +00) 这样3个区间， 每一个区间就对应着一个儿子指针（Child指针）。 在B+树中， Child指针一般指向一个页， 而在分型树中，每一个Child指针除了需要指向一个Node的地址(BlockNum)之外，还会带有一个Message Buffer (msg_buffer)， 这个Message Buffer 是一个先进先出(FIFO)的队列，用来存放Insert/Delete/Update/HotSchemaChange这样的更新操作。 

按照ft-index源代码的实现， 对ft-index中分型树更为严谨的说法是这样的: 

* 节点(block或者node, 在InnoDB中我们称之为Page或者页)是由一组有序的键值组成， 第一个键值设置为null键值， 表示负无穷大。 
* 节点分为两种类型，一种是叶子节点， 一种是非叶子节点。 叶子节点的儿子指针指向的是BasementNode,  非叶子节点指向的是正常的Node 。 这里的BasementNode节点是存放的是一个K-V键值对， 也就是说最后所有的查找操作都需要定位到BasementNode才能成功获取到数据(Value)。这一点也和B+树的LeafPage类似， 数据(Value)都是存放在叶子节点， 非叶子节点用来存放键值(Key)做索引。  当叶子节点加载到内存后，为了快速查找到BasementNode中的数据(Value)， ft-index会把整个BasementNode中的key-value都转换为一棵弱平衡二叉树， 这棵平衡二叉树有一个很逗逼的名字，叫做[替罪羊树](https://en.wikipedia.org/wiki/Scapegoat_tree)， 这里不再展开。
*  每个节点的键值区间对应着一个儿子指针(Child Pointer)。 每个儿子指针携带着一个[MessageBuffer](https://github.com/Tokutek/ft-index/blob/master/ft/msg_buffer.cc)， MessageBuffer是一个FIFO队列。用来存放Insert/Delete/Update/HotSchemaChange这样的更新操作。儿子指针以及MessageBuffer都会序列化存放在Node的磁盘文件中。
*  每个非叶子节点(Non Leaf Node)儿子指针的个数必须在[fantout/4,  fantout]这个区间之内。 这里fantout是分型树（B+树也有这个概念）的一个参数，这个参数主要用来维持树的高度。当一个非叶子节点的儿子指针个数超过fantout/4 ， 那么我们认为这个节点的太空虚了，需要和其他节点合并为一个节点(Node Merge)， 这样能减少整个树的高度。当一个非叶子节点的儿子指针个数超过fantout， 那么我们认为这个节点太饱满了， 需要将一个节点一拆为二(Node Split)。 通过这种约束控制，理论上就能讲磁盘数据维持在一个正常的相对平衡的树形结构，这样可以控制插入和查询复杂上限。 

> 注意： 在ft-index实现中，控制树平衡的条件更加复杂， 例如除了考虑fantout之外，还要保证节点总字节数在[NodeSize/4, NodeSize]这个区间， NodeSize一般为4M ，当不在这个区间时， 需要做对应的合并(Merge)或者分裂(Split)操作。
 
### 分型树的Insert/Delete/Update实现

在前文中，我们说到分型树是一种写优化的数据结构， 它的写操作性能要优于B+树的写操作性能。 那么它究竟如何做到更优的写操作性能呢？ 

首先， 这里说的写操作性能，指的是随机写操作。 举个简单例子，假设我们在MySQL的InnoDB表中不断执行这个SQL语句： insert into sbtest  set x = uuid()， 其中sbtest表中有一个唯一索引字段为x。 由于uuid()的随机性，将导致插入到sbtest表中的数据散落在各个不同的叶子节点(Leaf Node)中。 在B+树中， 大量的这种随机写操作将导致LRC-Cache中大量的热点数据页落在B+树的上层(如下图所示）。这样底层的叶子节点命中Cache的概率降低，从而造成大量的磁盘IO操作， 也就导致B+树的随机写性能瓶颈。但B+树的顺序写操作很快，因为顺序写操作充分利用了局部热点数据， 磁盘IO次数大大降低。 

<img src="/images/tokudb/innodb-lru-cache-dist.png" width="100%"> 


下面来说说分型树插入操作的流程。 为了方便后面描述，约定如下： 

a.   我们以Insert操作为例， 假定插入的数据为(Key, Value);   
b.  下文说的 __加载节点(Load Page)__，都是先判断该节点是否命中LRU-Cache。仅当缓存不命中时， ft-index才会通过seed定位到偏移量读取数据页到内存;  
c.   为体现核心流程， 我们暂时不考虑崩溃日志和事务处理。 

详细流程如下： 

1.   加载Root节点； 
2.   判断Root节点是否需要分裂(或合并)，如果满足分裂条件，则分裂Root节点。 具体分裂Root节点的流程，感兴趣的同学可以开开脑洞。 
3.   当Root节点不是叶子节点上时， 通过二分搜索找到Key所在的键值区间Range，将(Key, Value)包装成一条消息(Insert, Key, Value) ， 放入到键值区间Range对应的Child指针的Message Buffer中。 
4.   当Root节点是叶子节点时， 将消息(Insert, Key, Value)  应用到BasementNode上， 也就是插入(Key, Value)到BasementNode中。 

这里有一个非常诡异的地方，在大量的插入（包括随机和顺序插入）情况下， Root节点会经常性的被撑饱满，这将会导致Root节点做大量的分裂操作。然后，Root节点做了大量的分裂操作之后，产生大量的height=1的节点， 然后height=1的节点被撑爆满之后，又会产生大量height=2的节点， 最终树的高度越来越高。 这个诡异的之处就隐藏了分型树写操作性能比B+树高的秘诀： 每一次插入操作都落在Root节点就马上返回了， 每次写操作并不需要加载到树形结构最底层的BasementNode， 这样会导致大量的热点数据集中落在在Root节点的上层(此时的热点数据分布图类似于上图)， 从而充分利用热点数据的局部性，大大减少了磁盘IO操作。

Update/Delete操作的情况和Insert操作的情况类似， 但是需要特别注意的地方在于，由于分型树随机读性能并不如InnoDB的B+树（后文会详细描述）。因此，Update/Delete操作需要细分为两种情况考虑，这两种情况测试性能是可能差距巨大： 

* 覆盖式的Update/Delete (overwrite)。 也就是当key存在时， 执行update/Delete； 当key不存在时，不做任何操作，也不需要报错。 
* 严格匹配的Update/Delete。 当key存在时， 执行update/delete ; 当key不存在时， 需要报错给上层应用方。 在这种情况下，我们需要先查询key是否存在于ft-index的basementnode中，于是Point-Query默默的拖了Update/Delete操作的性能后退。  

此外，ft-index为了提升顺序写的性能，对顺序插入到操作了做了一些优化，例如[顺序写加速](http://www.kancloud.cn/taobaomysql/monthly/67144), 这里不再展开。

### 分型树的Point-Query实现

在ft-index中， 类似于select * from table where id = ? （其中id是索引）的查询语句称之为Point-Query； 类似于select * from table where id >= ? and id <= ? （其中id是索引）的查询语句称之为Range-Query。 上文已经提到， Ponit-Query读操作性能并不如InnoDB的B+树， 这里详细描述Point-Query的相关流程。  （这里假设要查询的键值为Key）

1.  加载Root节点，通过二分搜索确定Key落在Root节点的键值区间Range, 找到对应的Range的Child指针。 
2.  加载Child指针对应的的节点。 若该节点为非叶子节点，则继续沿着分型树一直往下查找，一直到叶子节点停止。 若当前节点为叶子节点，则停止查找。

查找到叶子节点后，我们并不能直接返回叶子节点中的BasementNode的Value给用户。 因为分型树的插入操作是通过消息(Message)的方式插入的， 此时需要把从Root节点到叶子节点这条路径上的所有消息依次apply到叶子节点的BasementNode。 待apply 所有的消息完成之后，查找BasementNode中的key对应的value，就是用户需要查找的值。 

分型树的查找流程基本和 InnoDB的B+树的查找流程类似， 区别在于分型树需要将从Root节点到叶子节点这条路径上的messge buffer都往下推(下推的具体流程请参考代码，这里不再展开)，并将消息apply到BasementNode节点上。注意查找流程需要下推消息， 这可能会造成路径上的部分节点被撑饱满，但是ft-index在查询过程中并不会对叶子做分裂和合并操作， 因为ft-index定下的设计原则是： Insert/Update/Delete操作负责节点的Split和Merge, Select操作负责消息的延迟下推(Lazy Push)。 这样，分型树就将Insert/Delete/Update这类更新操作通过未来的Select操作应用到具体的数据节点，从而完成更新。

### 分型树的Range-Query实现
下面来介绍Range-Query的查询实现。简单来讲， 分型树的Range-Query基本等价于进行N次Point-Query操作，操作的代码也基本等价于N次Point-Query操作的代码。  由于分型树在上层节点的msg_buffer中存放着BasementNode的更新操作，因此我们在查找每一个Key的Value时，都需要从根节点查找到叶子节点， 然后将这条路径上的消息apply到basenmentNode的Value上。 这个流程可以用下图来表示。 

![Alt txt](/images/tokudb/ft-index-push-down.png)

但是在B+树中， 由于底层的各个叶子节点都通过指针组织成一个双向链表， 结构如下图所示。 因此，我们只需要从跟节点到叶子节点定位到第一个满足条件的Key,  然后不断在叶子节点迭代next指针，即可获取到Range-Query的所有Key-Value键值。因此，对于B+树的Range-Query操作来说，除了第一次需要从root节点遍历到叶子节点做随机写操作，后继数据读取基本可以看做是顺序IO。

![Alt txt](/images/tokudb/innodb-index-search.png)
 
### 总结 
 
本节从时间复杂度以及实现角度，对比ft-index和B+树两种索引结构的优缺点。
 
### 参考资料

1.  https://github.com/Tokutek/ft-index
2.  https://en.wikipedia.org/wiki/Fractal_tree_index
3.  https://www.percona.com/about-percona/newsroom/press-releases/percona-acquires-tokutek
4. [《MySQL技术内部：InnoDB存储引擎》 by 姜承尧](http://book.douban.com/subject/5373022/)
5. https://en.wikipedia.org/wiki/Scapegoat_tree
6. https://en.wikipedia.org/wiki/Order-maintenance_problem
7. [Tokutek团队讲解Fractal Tree的相关PPT](http://pan.baidu.com/s/1bnFkCEV)
