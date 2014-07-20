---
layout: post
title: "谈谈Redis字典的实现"
description: ""
category: 
tags: [algorithm]
---

####声明：本文版权归OpenInx所有，博客地址：http://openinx.github.io
 
###Hash表（Hash Table）

hash表实际上由size个的桶组成一个桶数组table[0...size-1] 。当一个对象经过哈希之后，得到一个相应的value , 于是我们把这个对象放到桶table[ value ]中。当一个桶中有多个对象时，我们把桶中的对象组织成为一个链表。这在冲突处理上称之为拉链法。

###负载因子（load factor）

假设一个hash表中桶的个数为 size , 存储的元素个数为used .则我们称 used / size 为负载因子loadFactor . 一般的情况下，当`loadFactor<=1`时，hash表查找的期望复杂度为O(1). 因此，每次往hash表中添加元素时，我们必须保证是在`loadFactor<1`的情况下，才能够添加。

###容量扩张（Expand）& 分摊转移

当我们添加一个新元素时，一旦loadFactor大于等于1了，我们不能单纯的往hash表里边添加元素。因为添加完之后，loadFactor将大于1，这样也就不能保证查找的期望时间复杂度为常数级了。这时，我们应该对桶数组进行一次容量扩张，让size增大 。这样就能保证添加元素后 used / size 仍然小于等于1 ， 从而保证查找的期望时间复杂度为O(1).但是，如何进行容量扩张呢？ C++中的vector的容量扩张是一种好方法。于是有了如下思路 ：　

Hash表中每次发现`loadFactor==1`时，就开辟一个原来桶数组的两倍空间（称为新桶数组），然后把原来的桶数组中元素全部转移过来到新的桶数组中。注意这里转移是需要元素一个个重新哈希到新桶中的，原因后面会讲到。

这种方法的缺点是，容量扩张是一次完成的，期间要花很长时间一次转移hash表中的所有元素。这样在hash表中`loadFactor==1`时，往里边插入一个元素将会等候很长的时间。
redis中的dict.c中的设计思路是用两个hash表来进行进行扩容和转移的工作：当从第一个hash表的`loadFactor=1`时，如果要往字典里插入一个元素，首先为第二个hash表开辟2倍第一个hash表的容量，同时将第一个hash表的一个非空桶中元素全部转移到第二个hash表中，然后把待插入元素存储到第二个hash表里。继续往字典里插入第二个元素，又会将第一个hash表的一个非空桶中元素全部转移到第二个hash表中，然后把元素存储到第二个hash表里……直到第一个hash表为空。

这种策略就把第一个hash表所有元素的转移分摊为多次转移，而且每次转移的期望时间复杂度为O(1)。这样就不会出现某一次往字典中插入元素要等候很长时间的情况了。

为了更深入的理解这个过程，先看看在dict.h中的两个结构体：


```c
    typedef struct dictht {
        dictEntry **table;
        unsigned long size;
        unsigned long sizemask;
        unsigned long used;
    } dictht;

    typedef struct dict {
        dictType *type;
        void *privdata;
        dictht ht[2];
        int rehashidx; /* rehashing not in progress if rehashidx == -1 */
        int iterators; /* number of iterators currently running */
    } dict;
```


dictht指的就是上面说的桶数组，size用来表示容量，一般为2^n ，sizemask（一般为2^n-1,二进制表示为n个1）用来对哈希值取模 , used表示hash表中存储了多少个元素。
dict表示字典，由两个桶数组组成，type是一些函数指针（哈希函数及key，value的一些处理函数）。

###d->rehashidx
 
这个变量的理解很关键：

`d->rehashidx` 表明了新元素到底是存储到桶数组0中，还是桶数组1中，同时指明了d->h[0]中到底是哪一个桶转移到`d->h[1]`中。
当`d->rehashidx==-1`时，这时新添加的元素应该存储在桶数组0里边。
当`d->rehashidx!=-1` 时，表示应该将桶数组0中的第一个非空桶元素全部转移到桶数组1中来(不妨称这个过程为桶转移，或者称为rehash)。这个过程必须将非空桶中的元素一个个重新哈希放到桶数组1中，因为d->h[1]->sizemask已经不同于d->h[0]->sizemask了。这时新添加的元素应该存储在桶数组1里边，因为此刻的桶数组0的loadFactor为1 ，而桶数组1的loadFactor小于1 。
 
当发现桶数组0中的元素全部都转移到桶数组1中，即桶数组0为空时。释放桶数组0的空间，把桶数组0的指针指向桶数组1。将`d->rehashidx`赋值为-1 ， 这样桶数组1就空了，下次添加元素时，仍然添加到桶数组0中。直到桶数组0的元素个数超过桶的个数，我们又重新开辟桶数组0的2倍空间给桶数组1 ， 同时修改d->rehashidx=0，这样下次添加元素是就添加到桶数组1中去了。
 
值得注意的是，在每次删除、查找、替换操作进行之前，根据`d->rehashidx`的状态来判断是否需要进行桶转移。这可以加快转移速度。
 
下面是一份精简的伪代码，通过依次插入element[1..n]这n个元素到dict来详细描述容量扩张及转移的过程：


    //初始化两个hash表
    d->h[0].size = 4 ; d->h[1].used = 0 ;  //分配四个空桶
    d->h[1].size = 0 ; d->h[1].used = 0 ;  //初始化一个空表
    
    for(i = 1 ; i <= n ; ++ i){
        if( d->rehashidx !=-1 ){
                    if(d->h[0]->used != 0){
                       把 d->h[0]中一个非空桶元素转移（重新hash）到 d->h[1]中
                       // 上一步会使得:
                       // d->h[0]->used -= 转移的元素个数 
                       // d->h[1]->used += 转移的元素个数 ；
                       把 element[i] 哈希到 d->h[1]中  ;  // d->h[1]->used ++ 
                    }else{
                       //用桶数组1覆盖桶数组0；
                       //赋值前要释放d->h[0]的空间，赋值后重置d->h[1])
                       d->h[0] = d->h[1] ; 
                       d->rehashidx = -1 ; 
                       把element[i]哈希到d->h[0]中；// d->h[0]->used ++ ; 
                    }
        }else if( d->h[0]->used >= d->h[0]->size )
                    d->h[1] = new bucket[2*d->h[0]->size ];    
                    // d->h[0]->size 等于d->h[0]->size的2倍 
                    把element[i]哈希到d->h[1]中 ;  // d->h[1]->used ++ 
                    d->rehashidx = 0 ;                             
        }else{
                    把element[i]哈希到d->h[0]中;  // d->h[0]->used ++ 
        }
    }



###字典的迭代器（Iterator）
 
分为安全迭代器( safe Iterator )和非安全迭代器 。

安全迭代器能够保证在迭代器未释放之前，字典两个hash表之间不会进行桶转移。

桶转移对迭代器的影响是非常大的，假设一个迭代器指向d->h[0]的某个桶中的元素实体，在一次桶转移后，这个实体被rehash到d->h[1]中。而在d->h[1]中根本不知道哪些元素被迭代器放过过，哪些没有被访问过，这样有可能让迭代器重复访问或者缺失访问字典中的一些元素。所以安全迭代器能够保证不多不少不重复的访问到所有的元素（当然在迭代过程中，不能涉及插入新元素和删除新元素的操作）。
