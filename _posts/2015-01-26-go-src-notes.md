---
layout: post
title: "Golang Src Notes"
description: "golang src notes"
category: 
tags: [golang]
---

最近打算把Golang的src自带的依赖包都阅读下。　分类如下: 

## 基本算法和数据结构: 

1. sort
2. strings
3. bytes
4. container　包括heap, list, ring三种
5. index  包括后缀数组
6. hash 包括crc32, crc64,adler32，　[Fowler–Noll–Vo hash function](http://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function)四类checksum算法
7. strconv　从字符串中解析数值类型
8. bufio 带缓冲的io

## 信息编码,加密算法，压缩算法:

1. archive 归档算法. 包括tar和zip两类算法
2. compress 压缩算法: 包括bzip2, flate, gzip, lzw, zlib四种算法
3. crypto 加密算法: aes, cipher, des, dsa, ecdsa, elliptic, hmac, md5, rand, rc4, rsa, sha1, sha256, sha512,  subtle, tls, 509
4. unicode utf8, ut16编码算法

## 数学

1. math/big 实现大整数的加，减，乘，除，左移，右移，　取模。
2. math/cmplx 复数的abs, asin, log, pow, sin, sqrt, tan等操作
3. math/rand 随机数生成算法
4. math/ 常见的abs,acos, asin, atan, atan2, exp, floor, log, mod, pow, sin, sqrt, tan等函数的实现


## 语言特性及工具

1. cmd
2. debug
3. errors
4. expvar
5. flag 编写命令行参数参数选项工具包
6. fmt 格式化输入输出
7. go 　golang的词法解析，抽象语法树 
8. reflect 
9. regexp 正则表达式
10. log 日志包
11. testing 测试框架, 和benchmark测试框架


## 操作系统相关

1. time　时间操作
2. syscall
3. sync 包括cond, mutex, once, race, waitgrou等goroutine同步工具, 和基本数据类型的compareAndSwap的原子操作.
4. runtime 
5. os  包括信号量，终端命令,　文件描述符等操作。　

## 网络相关

1. html. 主要用于做页面渲染填充
2. net,  http客户端和服务端，　mail协议, jsonrpc框架, smtp客户端, URL解析工具, socket, dns协议, 　interface网卡，ip协议，　mac地址, tcp, udp.



## 图像处理

1. image　Point, Rectangle, pixel像素作图，png, jpeg, gif格式解析, RGB像素


## 数据库接口

1. database 数据库层接口, 在此基础上可以实现MySQL, SQLite, Oracle, PostgreSQL等数据库的驱动包


