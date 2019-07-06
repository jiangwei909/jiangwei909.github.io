---
title: 如何让nginx支持https
date: 2017-06-16 09:14:21 +0800
categories: 计算机技术
tags:
 - nginx
 - ssl
 - https
---

https会导致站点性能下降，但如果安全摆在首位，或者你是土豪，也可以启用https，倍儿有面子。接下来就介绍一下如何在nginx中启用https
<!-- more -->

1. 创建网站证书存放目录

		$ mkdir /usr/local/nginx/conf/ssl
		$ cd /usr/local/nginx/conf/ssl

2. 制作CA证书

		$ openssl genrsa -des3 -out ca.key 2048
		$ openssl req -new -x509 -days 7305 -key ca.key -out ca.crt

3. 生成nginx服务器所需证书，并用CA签名

		$ openssl genrsa -des3 -out client.key 1024
		$ openssl req -new -key client.key -out client.csr
		$ openssl x509 -req -in client.csr -out client.pem -signkey client.key -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650

4. 查看证书文件

		$ pwd
		/usr/local/nginx/conf/ssl
		$ ls
		ca.crt  ca.key  ca.srl  client.csr  client.key  client.pem

5. 修改nginx配置
在配置文件中增加

		ssl                  on;
		ssl_certificate      /usr/local/nginx/conf/ssl/client.pem;
		ssl_certificate_key  /usr/local/nginx/conf/ssl/client.key;
		 
6. 重新启动nginx
   
		$nginx -s reload
   
哦，表面上一切正常，可是如果查看日志的话，你会发现类似下面的错误

	2500#0: SSL_CTX_use_PrivateKey_file("/usr/share/nginx/ssl/client.key") failed (SSL: error:0906406D:PEM routines:PEM_def_callback:problems getting password error:0906A068:PEM routines:PEM_do_header:bad password read error:140B0009:SSL routines:SSL_CTX_use_PrivateKey_file:PEM lib)

怎么回事？

原来私钥证书的密码问题，解决方法如下：

	$openssl rsa -in client.key -out unclient.key
	
输入俩次私钥的密码：123456。这里假定你刚开始设定的密码是123456。 把unclient.key 文件修改为client.key。 重启nginx 问题解决。

整个过程很简单吧。什么？nginx,我的可是apache. -_-~

