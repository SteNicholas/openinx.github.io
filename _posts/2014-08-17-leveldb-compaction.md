---
layout: post
title: "LevelDB-Compaction"
description: ""
category:
tags: [ leveldb ]
---


### leveldb基本约束
在默认options下，leveldb的一些基本约束:  
1. leveldb的level有0,1,2,3,4,5,6共7个取值;  
2. 第0层的sstable在4M左右;  
3. 第i(i>0)层的sstable每个sstable最大空间不超过2M;  
4. 第0层的sstable理想的情况是4个，尽量控制在8个以内，最大值不超过12;  
5. 第i(i>0)层的所有sstable所占存储空间之和控制在`10^i M`左右;  
这里说的_控制_不是指严格控制，而是总体上大致控制;  

### Compaction定义
* minor compaction  
从内存中拿出一个immtable，直接dump成sstable文件，然后根据_一定的策略_放到第i(i>=0)层。记_策略函数_为 PickLevelForMemTableOutput().
* majar compaction  
从第i(i>=0)层按照_估价函数_取出一个或多个sstable,这些sstable集合记为up(i)集合。找出第i+1层与up(i)集合有overlap的sstable，记为down(i)集合。将up(i),down(i)两个集合的所有sstable做多路归并排序之后，导出的sstable全部放在i+1层。这个过程称为majar compaction. 记计算up(i)集合的估价函数为PickCompaction(i).

### Minor Compaction触发的条件
以下几个条件同时满足时，才会触发Minor Compaction:  
1. 在调用put/delete API时，发现memtable的使用空间超过4M了；  
2. 当前的immtable已经被dump出去成sstable. 也就是immtable=NULL  
在上面的两个条件同时满足的情况下，会阻塞写线程，把memtable移到immtable。然后新起一个memtable，让写操作写到这个memtable里。最后将imm放到后台线程去做compaction.

### Majar Compaction触发的条件
以下任一条件满足时，都会触发Major Compaction:  
1. 调用CompactRange这个API，手动触发compaction;  
2. 调用Get这个API的过程中，发现seek的第一个sstable的AllowedSeek消耗完了;  
3. 第0层的sstable超过8个;  
4. 第i(i>0)层的所有sstable的占用空间超过`10^i M`;  

其中第4点一般是在第i层做了一次compaction之后，发现i+1层的不满足_leveldb基本约束5_了，导致再做一次compaction.

### Minor Compaction流程

```cpp
1. sstable = MemtableDumpToSSTable(immtable);
2. i = PickLevelForMemTableOutput(sstable);
3. place SSTable to level i;
3. edit= getVersionEdit();
4. updateVersionSet();
```

其中层次选择函数`PickLevelForMemTableOutput()`如下：

```cpp
int PickLevelForMemTableOutput(sst){
    if( (sst overlap with Level[0])  OR (sst overlap with level[1])) 
        return 0;
    else{
        overlapBytes := calculateOverlapBytes(sst, level[2]);
        if( overlapBytes > 20M ) 
           return 0 ; 
        else 
           return 1 ;
    }
}
```

### Majar Compaction流程

* `MajarCompaction()`  

```cpp
c, i := PickCompaction(); // i is level
if( up(i).size() == 1  AND down(i).size() == 0) {  // down(i) is empty set.
   overlapBytes := calculateOverlapBytes(up(i), Level[i+2]);
   if( overlapBytes <= 20M ){
      Just place up(i) to (i+1) level. 
      return;
   }
}
DoCompactionWork;  // 每次合并的数据量在26M左右。
edit = updateEdit();
updateVersionSet(edit);
```

* `DoCompactionWork(up(i), down(i))`

```cpp
iter: = MergeIterator(up(i), down(i));
sst := NewSStable();
while(iter.Next()){
   sst.Add(iter);
   if( (sst.bytesSize() > 2M) OR calculateOverlapBytes(sst, Level[i+2]) > 20M){
       Place sst to level i+1;
       sst := NewSSTable();
   }
}
Place sst to level i+1;
```

### Why Compaction? 

`miniorCompatcion()`与`majarCompaction()`其实在维护一个约束： _参与compaction的数据来量控制在25M～26M左右_。  

为什么是25～26M呢？我认为是Leveldb期望的情况是，第`i(i>0)`层有`10^i`这个数据级的个数的sst, 假设有key的范围在`[1..10^6]`, 那么在key值较均匀的情况下，散落在第i层的每个sst的key的数据量应该为`10^(6-i)`, 那么第i层的每个sst与第i+1层的sst的重叠数据量基本上会在10个左右。这样的话，可以保证，每次做compactioin的数据量基本达到一个可控的范围之内，第i层1个sst加上第i+1层的10个sst,加上第i+1层的2个端点，共13个sst, 每个sst的数据上限是2M,那么总共数据量在26M左右。  

但是为什么需要做compaction呢？  

按照leveldb的设计，Write操作只需要写内存加上顺序写日志。假设在不做compaction的情况下，每次memtable写满了，就会直接dump到level-0。这样的话,level-0层的sst会越积越多，最后达到很庞大的一个数字，比如`10^6`。但是，第0层的任意两个sst的数据都是有可能有重叠区间的，那这样做read操作就痛苦了，一旦发现要读取的key在第0层的某个sst的区间内，就要去seek对应的block，把数据读出来，做二分查找，看能不能找到。想象一下假设`10^6`个sst,有`10^3`个sst的区间包含了key，那么一次读操作要做`10^3`次seek(当然这是在不考虑LRU-Cache的情况下),那read要简直慢到了不可想像的地步。  

现在有了compaction可以把level-0层的sst控制在一个小量的范围之内，同时在将非0层的sst整理成一个个区间互不重叠的sst。这样的话，做GET操作时，在第0层搜索的开销可以控制，同时，保证在第i(i>0)层最多进行一次随机IO就可以准确判断数据是否在该层次之内。所以，可以得出一个结论就是：_compaction操作是为了提高Get的性能_。  

还有一个问题是，后台的compaction速度可能远远跟不上write的速度，那么，照样在第0层会产生大量的sst,导致Get操作性能爆差。在此，leveldb发现0层的sst的数量达到8个时，对每次写操作进行休眠，控制写频率;当0层的sst超过12个时，直接阻塞write操作，等待compaction完成。所以，compaction操作是在牺牲Write操作高效率性能前提下，提高了下GET操作的性能。  

做compaction操作不能破坏数据均匀性(我认为数据均匀就是： 第i(i>0)层的每一个sst与第i+1层的sst的overlap的sst个数控制在10个以内。或者这么定义: 假设操作的所有key在[smallest,largets]这个区间，那么第i(i>0)层的每个sst应该均分这个区间的所有key)，因为不均匀的数据分布，会导致接下来的compaction耗费极大。  

总之, compaction是牺牲了write性能，提高get性能，然后compaction又做了一个自我平衡。leveldb就是各种均衡的结果。  

以上纯属SB在YY。  

