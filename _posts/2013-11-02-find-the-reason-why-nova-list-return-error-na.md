---
layout: post
title: "Find the reason why 'nova list' return 'Error: n/a'"
description: ""
category: 
tags: [openstack]
---
{% include JB/setup %}

今天安装了一下Openstack的G版本，安装好Nova之后，发现一个问题, 当用`nova list`
去测试验证是否通过时，发现老是报：

    ERROR: n/a (HTTP 401)

打开用nova --debug list测试一下发现, 第一次去Keystone请求Token的是正确返回的。

    curl -i http://127.0.0.1:35357/v2.0/tokens -X POST -H "Content-Type: application/json" -H "Accept: application/json" -H "User-Agent: python-novaclient" -d '{"auth": {"tenantName": "demo", "passwordCredentials": {"username": "admin", "password": "secrete"}}}'

但是，用返回的Token去请求：

    curl -i http://127.0.0.1:8774/v2/30c7c675fe81469a853564ec17f1818f/servers/detail

时报出了如下错误：

    reply: 'HTTP/1.1 401 Unauthorized\r\n'
    header: Www-Authenticate: Keystone uri='http://127.0.0.1:35357'
    header: Content-Length: 276
    header: Content-Type: text/plain; charset=UTF-8
    header: Date: Sat, 02 Nov 2013 09:49:44 GMT
    RESP:{'date': 'Sat, 02 Nov 2013 09:49:44 GMT', 'status': '401', 'content-length': '276', 'content-type': 'text/plain; charset=UTF-8', 'www-authenticate': "Keystone uri='http://127.0.0.1:35357'"} 401 Unauthorized
    
    This server could not verify that you are authorized to access the document you requested. Either you supplied the wrong credentials (e.g., bad password), or your browser does not understand how to supply the credentials required.
    
     Authentication required  
    
    DEBUG (shell:534) n/a (HTTP 401)
    Traceback (most recent call last):
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/shell.py", line 531, in main
        OpenStackComputeShell().main(sys.argv[1:])
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/shell.py", line 467, in main
        args.func(self.cs, args)
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/v1_1/shell.py", line 694, in do_list
        utils.print_list(cs.servers.list(search_opts=search_opts), columns,
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/v1_1/servers.py", line 309, in list
        return self._list("/servers%s%s" % (detail, query_string), "servers")
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/base.py", line 62, in _list
        _resp, body = self.api.client.get(url)
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/client.py", line 239, in get
        return self._cs_request(url, 'GET', **kwargs)
      File "/home/hz/env/lib/python2.7/site-packages/novaclient/client.py", line 236, in _cs_request
        raise ex

乍一看，应该是nova的/etc/nova/nova.conf的配置文件的keystone认证那一个块配置错了。
但是经过反复确认之后，发现/etc/nova/nova.conf里面那一块的配置文件并没有问题。

无奈之下，只好打开keystone的DEBUG日志。  发现了这么一段。

    2013-11-02 17:51:55    ERROR [keystone.common.wsgi] Could not find user: %SERVICE_USER%
    Traceback (most recent call last):                                             
      File "/usr/lib/python2.7/dist-packages/keystone/common/wsgi.py", line 266, in __call__
        result = method(context, **params)                                         
      File "/usr/lib/python2.7/dist-packages/keystone/token/controllers.py", line 79, in authenticate
        context, auth)                                                             
      File "/usr/lib/python2.7/dist-packages/keystone/token/controllers.py", line 292, in _authenticate_local
        raise exception.Unauthorized(e)                                            
    Unauthorized: Could not find user: %SERVICE_USER%             
    2013-11-02 17:51:55  WARNING [keystone.common.wsgi] Authorization failed. Could not find user: %SERVICE_USER% from 127.0.0.1

当然异常栈是自己在代码里面加了一行，才打出来的。 刚开始还以为 %SERVICE_USER% 是keystone作者为了
屏蔽掉用户的用户名和密码之类的敏感信息，故意不把用户名和秘密打印出来。 后面发现原来NOVA发请求
到Keystone带的参数就是%SERVER_USER%。

明显, NOva应该用nova自己的服务用户名以及自己的服务的密码去验证才对。怎么会用一个这么抽象的串丟给Kestone呢？


后面用反复查看配置文件，包括/etc/nova/nova.conf 和/etc/nova/api-paste.ini 。发现了api-paste.ini里面
居然有一段keystone的配置 。 现在贴出来如下：

    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    auth_host = 127.0.0.1
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = %SERVICE_TENANT_NAME%
    admin_user = %SERVICE_USER%
    admin_password = %SERVICE_PASSWORD%


然后，我将其改成：

    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    auth_host = 127.0.0.1
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name=service
    admin_user=nova
    admin_password=nova

再执行nova list ， 就不再报这个错误了。 


对此，我只能深深的吐槽:  

是Openstack里面的哪个SB决定把NOVA验证Keystone密码配置选项放到api-paste.ini这个文件里面 ? 
而且是在/etc/nova/nova.conf明明有keystone认证这一段的情况下。 难道一个验证的配置选想不能
直接就配一次吗， 这至少给我造成了极大的困惑，也浪费了我不少时间。
