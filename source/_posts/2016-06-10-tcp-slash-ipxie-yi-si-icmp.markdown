---
layout: post
title: "TCP/IP协议(四):ICMP"
date: 2016-06-10 19:21:47 +0800
comments: true
categories: network_protocols
---

##引言
ICMP(Internet Control Message Protocol)是介于网络层和传输层的协议，它的主要功能是传输网络诊断信息。ICMP报文通常被IP层或更高层协议(TCP或UDP)使用，不过，也有一些ICMP报文把差错报文返回给用户进程<!--more-->。  

ICMP传输的信息可以分为两类，一类是错误信息，这一类信息可用来诊断网络故障。我们已经知道，IP协议的工作方式是"Best effort",如果IP数据包没有被传送到目的地，或者IP包发生错误，IP协议本身不会做进一步的努力。但上游发送IP包的主机和接力的路由器并不知道下游发生了错误和故障，它们可能继续发送IP包。通过ICMP包，下游的路由器和主机可以将错误信息汇报给上游，从而让上游的路由器和主机进行调整。需要注意的是，ICMP只提供特定类型的错误汇报，它不能帮助IP协议成为可靠的协议。  
另一类信息是查询性质的，比如某台计算机询问路径上的每个路由器都是谁，然后各个路由器同样用ICMP包回答。  

##1.ICMP报文  
ICMP报文的格式如下所示：  

![icmp_datagram](http://7xn1yt.com1.z0.glb.clouddn.com/ICMP_datagram.png)

显然，ICMP报文的格式非常简单。类型虽然有8位，但是目前RFC只用到了15种类型，而代码是这15种类型中的子类型。  
检验和字段覆盖的是整个ICMP报文而不只是头部。  

下表是ICMP差错报文中的类型--代码--描述表： 
![icmp_type_code](http://7xn1yt.com1.z0.glb.clouddn.com/icmp_type_code.png)

图中最后一列表示这个ICMP报文是一份查询报文还是差错报文，因为对ICMP差错报文有时需要作特殊处理，比如在对ICMP差错报文进行响应时，永远不会生成另一份ICMP差错报文(如果没有这个限制规则，可能会遇到一个差错产生另一个差错的情况，从而导致死循环).

ICMP报文通常是由某个IP包触发的，这个触发IP包的头部和一部分数据会被包含在ICMP包的数据部分。ICMP协议是实现ping命令和traceroute命令的，这两个工具常用于网络排错。  

##2.常用的ICMP包类型  
###2.1 回音(Echo)  
回音(Echo)属于查询信息，ping命令就是利用了该类型的ICMP包。当使用ping命令的时候，将向目标主机发送Echo-询问类型的ICMP包，而目标主机在接收到该ICMP包之后，会回复Echo-回答类型的ICMP包，并将询问ICMP包包含在数据部分。ping命令是我们进行网络排查的一个重要工具，如果一个IP地址可以通过ping命令收到回复，那么其他的网络协议通信方式也很有可能成功。  

###2.2 源头冷却(Source Quench)
源头冷却(Source Quench)属于错误信息，如果某个主机快速的向目的地传送数据，而目的地主机没有匹配的处理能力，目的地主机可以向出发主机发生该类型的ICMP包，提醒出发主机放慢发送速度。

###2.3 目的地无法到达
目的地无法到达(Destination Unreachable)属于错误信息。如果一个路由器接收到一个没办法进一步接力的IP包，它会向源主机发送该类型的ICMP包。比如当IP包到达最后一个路由器，路由器发现目的主机宕机，就会向源主机发送目的地无法到达(Destination Unreachable)类型的ICMP包。目的地无法到达还可能有其他的原因，比如不存在接力路径，比如不被接收的端口号等等。

###2.4 超时  
超时(Time Exceeded)属于错误信息。IPv4中的Time to Live(TTL)和IPv6中的Hop Limit会随着经过的路由器而递减，当这个区域值为0时，就认为该IP包超时.Time Exceeded就是TTL减为0时的路由器发给源主机的ICMP包，通知它发生了超时错误。  

traceroute就利用了这种类型的ICMP包。traceroute命令用来发现IP接力路径上的各个路由器，它向目的地发送IP包，第一次的时候，将TTL设置为1，引发第一个路由器的Time Exceeded错误，这样，第一个路由器回复ICMP包，从而 使源主机知道途径的第一个路由器的信息。随后TTL被设置为2,3,4,...直到到达目的主机，这样，沿途的每个路由器都会向源主机发送ICMP包来汇报错误。traceroute将ICMP包的信息打印在屏幕上，就是接力路径的信息了。  

###2.5 重定向 
重定向属于错误信息。当一个路由器收到一个IP包，对照其routing table,发现自己不应该收到该IP包，它会向源主机发送重新定向类型的ICMP，提醒源主机修改自己的routing table.

###2.6 IPv6的Neighbor Discovery
ARP协议用于获取对应某个IP地址的MAC地址，但是，ARP协议只用于IPv4,IPv6并不使用ARP协议。实际上，IPv6包通过邻居探索(ND,Neighbor Discovery)来实现ARP的功能。ND的工作方式与ARP类似，但它基于ICMP协议，ICMP包有Neighbor Solicitation和Neighbor Advertisement类型，这两个类型分别对应ARP协议的询问和应答信息。  

##3.总结
ICMP协议是IP协议的排错帮手，它可以帮助人们及时发现IP通信中出现的故障。基于ICMP的ping和traceroute也构成了重要的网络诊断工具。然而，需要注意的是，尽管ICMP的设计是出于好的意图，但ICMP却经常被黑客借用进行网络攻击，比如利用伪造的IP包引发大量的ICMP回复，并将这些ICMP包导向受害主机，从而形成DoS攻击。而redirect类型的ICMP包可以引起某个主机更改自己的routing table,所以也被用作攻击工具。许多站点选择忽视某些类型的ICMP包来提高自身的安全性。  


