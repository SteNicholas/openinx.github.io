---
layout: post
title: "TokuDB的索引结构: ft-index"
description: ""
category: 
tags: [MySQL, Database, TokuDB, Storage Engine]
---


在数据库中，作为索引结构的数据结构一般有: B/B+树, Hash索引, LSM-Tree索引, Fractal-Tree索引。 其中B/B+树和Hash索引是两种应用最大的索引结构， 对于B/B+树而言， 就有InnoDB存储引擎/MyISAM存储引擎／XFS文件系统／Berkeley DB索引结构／sqlite数据库。 Hash索引时一种高效快速的索引结构, 但是由于只支持Point Query， 不支持Range Query，因此不及B/B+树应用广泛， 我已知的数据库产品有Kyotocabinet, MySQL也实现了Hash索引。LSM-Tree是针对写优化的数据结构，一般来讲写操作的性能要比读性能好, 常用的数据库存储引擎有LevelDB, RocksDB, HBase, Cassandra， 这类数据库一般用于写压力远大于读压力的业务，比如大数据分析／日志存储等等。 Fractal-Tree则相对B/B+树和LSM树，显得更为中庸，也就是说Fractal-Tree的写性能要比LSM Tree差，但是比B+树性能要好， 常用的数据库存储引擎有Ft-Index（由tokumx开发的一款分型树k-v索引存储引擎）。

对这几类作为索引的数据结构，我做简单的归纳如下： 



<table>
<tr>
	<td>索引结构 </td>
	<td>B+树</td>
	<td>Hash索引</td>
	<td>LSM-Tree索引</td>
	<td>Fractal-Tree索引</td>
</tr>
<tr>
	<td>数据库存储引擎</td>
	<td>InnoDB, MyISAM, XFS, BerkeyleyDB, SQLite </td>
	<td>KyotoTreeDB, MySQL Hash Index</td>
	<td>LevelDB, RocksDB, HBase, Cassandra</td>
	<td>Ft-Index</td>
</tr>
<tr>
	<td>优点</td>
	<td>支持Point-Query, Range-Query， 顺序读性能好</td>
	<td></td>
	<td></td>
	<td></td>
</tr>
<tr>
	<td>缺点</td>
	<td>随机读性能差</td>
	<td></td>
	<td></td>
	<td></td>
</tr>
</table>



参考文献
-------
