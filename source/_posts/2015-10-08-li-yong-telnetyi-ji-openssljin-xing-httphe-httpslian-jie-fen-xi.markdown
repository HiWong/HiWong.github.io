---
layout: post
title: "利用telnet以及openssl进行http和https连接分析"
date: 2015-10-08 21:41:25 +0800
comments: true
categories: Network_protocols
---

在开发或者分析协议栈的过程中，常常需要抓包分析。Windows和Mac下可以用Wireshark,Fiddler等，但是Linux下最常用的是tcpdump. 如果只是简单的http(s)连接分析，则可以用telnet,openssl等。本文将介绍Linux下利用telnet以及openssl进行http和https连接分析。  

1.telnet  
telnet适用于http连接，下面以京东商城为例。<!--more-->  
首先在Terminal中输入以下命令：  

	$telnet www.jd.com 80

其中80是使用的端口。在建立连接之后，比如我们想查看进入首页的连接情况，输入以下命令：  

	$GET /index.html HTTP/1.1
     host:www.jd.com

之后回车两次，会出现以下数据返回:  

![telnet_jd](http://7xn1yt.com1.z0.glb.clouddn.com/telnet_jd.png)  

上图只是返回数据的一部分。    

比如我们要比较HTTP/1.1和HTTP/1.0的区别，则可分别输入GET /index.html HTTP/1.1和GET /index.hmtl HTTP/1.0然后查看结果，会发现HTTP/1.1在数据返回后Keep-Alive，但是HTTP/1.0则会断开连接。  

2.openssl  
以前大家测试ping和telnet时，常以www.baidu.com为例，但是现在百度已经不是采用http而是https了，如果还想测试其连接的话，不能再使用telnet，而要使用openssl了,如果所用Linux发行版没有openssl client的话，就要自己先安装。安装好之后，键入以下命令:  

	$openssl s_client -connect www.baidu.com:443  

然后会有以下返回：  
![openssl_01](http://7xn1yt.com1.z0.glb.clouddn.com/openssl_01.png)

![openssl_02](http://7xn1yt.com1.z0.glb.clouddn.com/open_ssl02.png)

显然，https连接建立需要先获取秘钥。然后，仍以index为例，输入以下命令：  
	
	$GET /index.html HTTP/1.1
	 host:www.baidu.com

然后回车两次可得到以下输出：  

![openssl_03](http://7xn1yt.com1.z0.glb.clouddn.com/openssl_03.png)

![openssl_04](http://7xn1yt.com1.z0.glb.clouddn.com/openssl_05.png)
