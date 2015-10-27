---
layout: post
title: "MySQL Proxy认证方式实现的一种思路"
description: "MySQL Proxy"
category: 
tags: [MySQL, Proxy]
---

### 背景

一谈到MySQL Proxy， 用户认证是一道无法越过的坎，尤其是实现和MySQL表现形式完全一致的Proxy时。 那么为什么MySQL的用户认真是一道无法越过的坎呢？ 为了便于更好的理解故事背景，我先简单介绍一下MySQL服务端和客户端的认证方式:


1. MySQL的客户端发起tcp连接到MySQL的服务器端。 
2. MySQL服务器端收到MySQL客户端的tcp请求之后，发送给客户端一个20字节的seed , 同时将seed保存在MySQL tcp连接的会话缓存中； 
3. 客户端收到seed之后， 使用客户端的username和password按照如下方式计算出一个签名: 

```
signature = sha1(password) xor sha1( seed +  sha1(sha1(password)) ); 
```

然后将计算出来的signature发送到MySQL服务端。 

4.  MySQL服务端收到客户端发过来的signature之后， 读取mysql.user表中的password的哈希值。 这里需要注意的是，mysql.user表中存放的事用户通过grant语句设置的密码的哈希值 。 下面的例子可以看出： 

mysql服务端计算密码的哈希算法为： `sha1(sha1(password))`

```
mysql> select password('123456'), upper(concat('*', sha1(unhex(sha1('123456'))))) ;
+-------------------------------------------+-------------------------------------------------+
| password('123456')                        | upper(concat('*', sha1(unhex(sha1('123456'))))) |
+-------------------------------------------+-------------------------------------------------+
| *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9       |
+-------------------------------------------+-------------------------------------------------+
1 row in set (0.00 sec)
```
 
 MySQL服务端拿到signature之后 ， 读取mysql.user表中的password的哈希值 ， 然后按照如下方式验证用户的signature是否正确； 

```
sha1Password = signature xor sha1( seed + mysql.user.passwordHashValue ); 
check_hash_value = sha1(sha1Password)
if check_hash_value == mysql.user.passwordHashValue ; then ; 
	return 'auth ok'
else:
	return 'auth failed'
```

这里主要用到一条性质： `a = a xor b xor b` 。 至此， MySQL完成了整个用户认证的流程。


这里， MySQL通过以下几点来保证用户密码的安全性：   

* 杜绝在mysql.user表中保存用户的明文密码， 这样计算mysql的整个库被人脱掉， 也不致于被人看到用户的明文密码。 
* 杜绝在TCP协议层传输明文密码， 这样能够避免别人通过抓取网络数据包来获取到用户的明文密码。 

这样也就最大限度的保护的用户的明文密码, 同时又实现了用户认证。此外，还有一个关键点，就是即便是mysql.user表中的password的哈希值被他人获取到了，用户也无法在不知道密码明文的情况下来伪造客户端签名来伪造客户端认证。因为在这种情况下， 为了计算签名，我们仍然需要`sha1(password)`这个值。 而这个值在没有任何办法可以获取到。这种情况是有可能发生的， 设想一下当用户A授权不当(一般就是授权太大)时，可能会导致用户A可以查看mysql.user表， 从而可以看到所有用户的密码的哈希值, 但即便是这样，用户A依然无法通过用户B在数据库中存放的密码的哈希值，来伪造B的签名信息痛过认证，然后借着B的名号干坏事情。 


### 


### 



### 参考资料: 

* [MySQL官方文档](https://dev.mysql.com/doc/internals/en/)
* [MySQL内核认证代码](https://github.com/mysql/mysql-server/blob/5.7/sql/auth/password.c)

