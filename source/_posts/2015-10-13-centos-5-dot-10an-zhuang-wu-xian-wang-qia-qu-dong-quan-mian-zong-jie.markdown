---
layout: post
title: "CentOS 5.10安装无线网卡驱动全面总结"
date: 2015-10-13 13:19:42 +0800
comments: true
categories: Linux
---
这几天因为想尝试一下CentOS，所以就在笔记本上安装了一个CentOS 5.10，但是安装完之后问题来了：它不像Ubuntu那样安装后之后就有相应的无线网卡驱动。所以如果要使用YUM安装或更新软件的话，第一件事就是安装无线网卡驱动。这中间的过程实在异常曲折，因为网络上很多人的文章是在已经能上网（比如通过有线上网）的前提下来安装无线网卡驱动，那自然简单许多。为了让后来者能更轻松地<!--more-->fix这个问题，特意写下本文。  

第一步：查看自己的无线网卡型号  

在root下输入以下命令即可：  

	lspci|grep Ethernet

我的显示的是05:00.0 Ethernet controller:Qualcomm Atheros:AR8151 v2.0 Gigabit Ethernet(revc0)
这说明我的网卡型号是AR8151  

一般来说像在Win系统下直接到它的官网获取驱动就可以，但是我上它的官网发现它竟然没有为Linux系统准备相应的驱动，主要原因可能是厂商觉得Linux的用户少，投入不值得吧。  

幸运的是，有很多热心的网友自己写出了相应的驱动，我在http://www.linuxidc.com/Linux/2012-11/75101.htm 这篇文章中找到了相应的驱动，再次感谢作者的无私贡献。  

如果有读者想要这个驱动的，在下面留下邮箱，我可以发给你。  

第二步：做好安装前的准备工作  

上面提到的那篇文章，其实省略了安装驱动前的准备工作，那就是先安装好kernel-headers和gcc编译器，否则解压后安装全出现makefile:61*** linux kernel source not found的错误。  

有人建议到CentOS的官网去找相应的安装包，其实也不是不可以，但是这样可能会出现微小的版本不兼容问题。其实完全没必要，因为这些都包含在安装系统所用的iso中，只要解压它，然后在CentOS下就可以找到kernel-headers-2.6.18-371.el5.i386.rpm,kernel-devel-2.6.18-371.el5.i686.rpm,gcc-4.1.2-54.el5.i386.rpm这些文件。当然，还有相应的库依赖，但是这些所需要的库也都在CentOS下。  

所以我们只需要用U盘或者光盘copy过去，然后挂载即可安装。下面是安装顺序：	  

	rpm -ivh cpp-4.1.2-54.el5.i386.rpm

    rpm -ivh kernel-headers-2.6.18-371.el5.i386.rpm

    rpm -ivh kernel-devel-2.6.18-371.el5.i686.rpm

    rpm -ivh glibc-headers-2.5-118.i386.rpm

    rpm -ivh glibc-devel-2.5-118.i386.rpm

    rpm -ivh libgomp-4.4.7-1.el5.i386.rpm

    rpm -ivh gcc-4.1.2-54.el5.i386.rpm

之所以需要安装gcc编译器，是因为所获取的驱动文件其实还是源文件，需要经过编译才能使用。

第三步：编译并安装无线网卡驱动。

将驱动文件解压放到/usr/local/src/nicdriver目录下面（如果没有这个目录就先mkdir)，然后再依次执行下面的命令：

   1. cd /usr/local/src/nicdriver
   2. make
   3. make install
   4. cd /lib/modules/2.6.18-194.el5PAE/kernel/drivers/net/atl1e(之所以要将目录切换到这里，是因为上面的操作，会在此目录下面生成一个atl1e.ko文件，这个文件正是我们所需要的)
   5. insmod atl1e.ko(在执行这一步的操作时，会显示“insmod: error inserting 'atl1e.ko': -1 File exists”的信息，不用理会，继续执行下面的命令)
   6. lsmod |grep atl1e(如果执行这一步的操作时，显示类似“atl1e 744000”的信息，表示已经成功完装了驱动)
   7. ifconfig -a(再次确认一下，如果在命令的输出中显示有“eth0”的字样，那就表示网卡已经正常了)

当完成以上的步骤之后，你就可以设置网卡的其它信息，如IP地址，DNS等。此时如果想要像Win那样设置无线网络，则只需要以下两个命令即可：

首先，开启NetworkManager的服务，在root用户下，执行

	chkconfig NetworkManager on

然后重启网卡

	service NetworkManager start

这样在图形界面的右上角就能够出现一个无线网络图标，点进去，即可搜到附近的网络信号。  

但是我个人还是更喜欢直接用命令行进行配置，这方面已经有非常详细的讲解，我就不重复造轮子了，想要用命令行进行网络配置的推荐下面这篇文章：http://blog.csdn.net/centre10/article/details/6769490

