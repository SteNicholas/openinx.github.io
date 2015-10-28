---
layout: post
title: "MySQL Proxy认证方式实现的一种思路"
description: "MySQL Proxy"
category: 
tags: [MySQL, Proxy]
---

### 背景

一谈到MySQL Proxy， 用户认证是一道无法越过的坎，尤其是实现和MySQL表现形式完全一致的Proxy时。 那么为什么MySQL的用户认真是一道无法越过的坎呢？ 为了便于更好的理解故事背景，我先简单介绍一下MySQL服务端和客户端的认证方式:


*  MySQL的客户端发起tcp连接到MySQL的服务器端。 
* MySQL服务器端收到MySQL客户端的tcp请求之后，发送给客户端一个20字节的seed , 同时将seed保存在MySQL tcp连接的会话缓存中； 
* 客户端收到seed之后， 使用客户端的username和password按照如下方式计算出一个签名: 

```
signature = sha1(password) xor sha1( seed +  sha1(sha1(password)) ); 
```

然后将计算出来的signature发送到MySQL服务端。 

*  MySQL服务端收到客户端发过来的signature之后， 读取mysql.user表中的password的哈希值。 这里需要注意的是，mysql.user表中存放的事用户通过grant语句设置的密码的哈希值 。 下面的例子可以看出： 

mysql服务端计算密码的哈希算法为： `sha1(sha1(password))`


```
mysql> select password('123456'); 
+-------------------------------------------+
| password('123456')                        |
+-------------------------------------------+
| *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
+-------------------------------------------+
1 row in set (0.00 sec)

mysql> select upper(concat('*', sha1(unhex(sha1('123456'))))); 
+-------------------------------------------------+
| upper(concat('*', sha1(unhex(sha1('123456'))))) |
+-------------------------------------------------+
| *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9       |
+-------------------------------------------------+
1 row in set (0.00 sec)
```
 
 MySQL服务端拿到signature之后 ， 读取mysql.user表中的password的哈希值 ， 然后按照如下方式验证用户的signature是否正确； 

```
sha1Password = signature xor sha1( seed + mysql.user.passwordHashValue ); 
check_hash_value = sha1(sha1Password)
if check_hash_value == mysql.user.passwordHashValue: 
	return 'auth ok'
else:
	return 'auth failed'
```

这里主要用到一条性质： `a = a xor b xor b` 。 至此， MySQL完成了整个用户认证的流程。


这里， MySQL通过以下几点来保证用户密码的安全性：   

* 杜绝在mysql.user表中保存用户的明文密码， 这样计算mysql的整个库被人脱掉， 也不致于被人看到用户的明文密码。 
* 杜绝在TCP协议层传输明文密码， 这样能够避免别人通过抓取网络数据包来获取到用户的明文密码。 

这样也就最大限度的保护的用户的明文密码, 同时又实现了用户认证。此外，还有一个关键点，就是即便是mysql.user表中的password的哈希值被他人获取到了，用户也无法在不知道密码明文的情况下来伪造客户端签名来伪造客户端认证。因为在这种情况下， 为了计算签名，我们仍然需要`sha1(password)`这个值。 而这个值在没有任何办法可以获取到。这种情况是有可能发生的， 设想一下当用户A授权不当(一般就是授权太大)时，可能会导致用户A可以查看mysql.user表， 从而可以看到所有用户的密码的哈希值, 但即便是这样，用户A依然无法通过用户B在数据库中存放的密码的哈希值，来伪造B的签名信息痛过认证，然后借着B的名号干坏事情。 


### 中间件认证 

常用的MySQL中间件的实现方式是， 中间件的连接管理分为前端连接(frontConnection)和后端连接(backendConnection)：

* 用户向proxy发起的连接叫做前端连接。 
** roxy向后端MySQL数据库发起的连接，叫做后端连接。*
在常用的Proxy，例如Cobar, MySQL-Proxy， 他们为了*上的简单， 采用把前端连接和后端连接的用户名和密码保存在本地的配置文件的方式来管理整个认证。其整个认证流程如下： 

* 客户端向Proxy发起TCP连接 
* Proxy接受客户但发起的TCP连接之后，返回给客户端一个20字节的seed，并在会话(Session)中保存该seed。
* 客户端收到Proxy的Seed之后， 计算签名值， 然后将签名发送给Proxy

```
*gnature = sha1(password) xor sha1(seed + sha1(sha1(password)))
``` 
* Proxy收到客户端的Signature之后，读取本地配置文件的密码password和会话保存的种子(seed)，重新计算一遍签名，然后判断Proxy计算的签名是否和客户端发送的签名一致。若一致则认证通过，若不一致则认证失败。

* Proxy读取后端连接对应的用户名和密码， 采用类似JDBC的方式来验证后端MYSQL数据库。 


那么在上述流程中，有几个问题需要注意： 

* 首先前端连接和后端连接的用户名和密码都是采用明文的方式保存在配置文件中。 对公有云的云端数据库来说，由于用户需要访问中间件，一旦中间件沦陷，那么可以获取到所有用户的明文密码，后果不堪设想。 但对于企业内部的私有云服务，这种方式其实并没有什么不妥，因为服务器部署在公司内部且不需要对外开放访问， 相对公有云服务来说比较安全。
* 当前端用户需要添加新的用户账号，或者密码时， 需要修改Proxy的本地配置文件的用户名和密码， 并重建加载配置文件。 这对云数据库服务来说，将管理流程复杂化了很多。 


### trick的思路

下面来介绍一种比较讨巧的方式： 
首先这种方式，不保存明文的用户名和密码， 其次在客户端的Session中保存需要保用户密码的sha1(password)值， 该值时一个保存在内存中的密码哈希值。 那么，整个认证的流程如下： 

* 客户端发起TCP连接
* Proxy服务端返回给客户端一个种子(Seed) , 该种子需要保存在客户端的会话缓存中。
* 客户端收到Seed之后，计算签名，然后发送签名给服务端
* (注意这里很关键)Proxy收到签名signature之后，可以计算出sha1(password)的值，并以此验证签名是否合法。计算方式如下： 

```
# 注意mysql.user.PasswordHashValue是存放在mysql.user表中的密码经过两次sha1之后的哈希值
sha1(password) = signature xor sha1(seed + mysql.user.PasswordHashValue)
# 验证签名是否合法
sha1sha1Password = sha1(sha1(password))
if sha1sha1Password == mysql.user.PasswordHashValue: 
	return 'auth ok'
else:
	return 'auth failed'
```

注意，我们在计算得到`sha1(password)`的值后，  必须将该值保存在session的缓存中， 那为啥要保存该值呢？ 因为有了该值，Proxy可以模拟前端连接向后端连接发起连接，并通过认证。 因为签名可以这样计算： 

`signature=sha1Password xor sha1(seed + sha1(sha1Password))`

* Proxy中间件根据session缓存中sha1Password值，向后端连接发起认证，并通过后端MySQL的认证。 



###  总结

采用最后一种方式有哪些好处呢? 

* 不需要保存前端用户的任何用户名和明文密码，保证了用户密码的安全性。 
* 当用户通过grant命令修改用户的密码时, Proxy不需要做额外的管理流程，只需将sql转发到后端服务即可。 
* Proxy不需要做额外的用户权限认证，因为后端的用户和前端的用户的认证信息一致，proxy只需要根据后端连接的返回信息直接返回给前端用户即可。 


PS.1: 这种实现方式比较trick的地方在于，在session中缓存sha1Password的值。 一方面，有了这个值是可以不需要密码就能通过后端认证的；另一方面，我们把这个值存放在本地内存中，也就保证了没有合适途径可以获取到该值，进而实施攻击。 

PS.2: 由于sha1Password存放在本地的缓存，那么当Proxy出现宕机之后怎么办？这个问题其实也比较好解释，由于Proxy进程已经挂了，那么客户端的tcp连接已经断开，需要重新发起TCP连接到Proxy，那么在重新建立MySQL连接的过程中，我们的Proxy可以在此获取到sha1Password的值，进而进行后端连接认证了。^_^


### 参考资料: 

* [MySQL官方文档](https://dev.mysql.com/doc/internals/en/)
* [MySQL内核认证代码](https://github.com/mysql/mysql-server/blob/5.7/sql/auth/password.c)

