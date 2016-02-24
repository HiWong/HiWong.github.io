---
layout: post
title: "Android Binder机制(1):Binder架构分析"
date: 2016-02-24 20:42:50 +0800
comments: true
categories: android_deep_analysis
---

从这篇博客开始，将进入Binder机制的分析系列，顺序是先讲解Binder机制的框架，理解了整体思想后，再深入分析各层的细节实现，最后会实现一个自己的本地服务。  

1.Binder的历史  
BeOS是Be公司在1991年开发的运行在BeBOX硬件上的一款操作系统，与同期的其他操作系统不同，它是一款基于GUI设计的操作系统。  

George Hoffman(在LinkedIn上可以找到这哥们，简直是活着的传奇)当时任Be公司的工程师，他启动了一个名为OpenBinder的项目，该项目的宗旨是研究一个高效的信号传递工具，允许软件的各个构成元素相互传递信号，从而使多个软件相互协作，共同形成一个软件系统。在Be公司被ParmSource公司收购后，OpenBinder由Dinnie Hackborn继续开发，后来成为管理ParmOS6 Cobalt OS的进程的基础。在Hackborn加入谷歌后，他在OpenBinder的基础上开发出了Android Binder(以下简称Binder)，用来完成Android的进程通信。  

Binder原本是IPC(Inter Process Communication)工具，但在Android中它主要用于支持RPC(Remote Procedure Call,远程调用)，使得当前进程调用另一个进程的函数时就像调用自身的函数一样轻松、简单<!--more-->。  

2.Linux内存空间与Binder Driver  

Android是基于Linux内核的操作平台，Android进程与Linux进程一样，它们只运行在进程固有的虚拟地址空间中。一个4GB的虚拟地址空间，其中3GB是用户空间，剩余的1G是内核空间(可通过内核设定修改).用户代码与相关库分别运行在用户空间的代码区域、数据区域，以及堆栈区域中，而内核空间中运行的代码则运行在内核空间的各个区域中。并且，进程具有各自独立的地址空间，单独运行。  

那么，一个拥有独立空间的进程如何向另一个进程传递数据呢？显然要通过两个进程共享的内核空间。虽然各个进程拥有各自独立的用户空间，但内核空间是共享的。从内核的角度看，进程不过是一个作业单位，虽然各进程的用户空间相互独立，但运行在内核空间中的任务数据、代码都是彼此共享的。  

事实上，Binder机制的架构如下所示：  

![binder_architecture](http://7xn1yt.com1.z0.glb.clouddn.com/binder_architecture.png)

下面是对各抽象层的说明：  

+ 服务层:该层包含一系列提供特定功能的服务函数。服务客户端实际上调用的是代理服务对象的服务函数，而实际调用是由Service Server进行的，即Service Server实际调用服务客户端请求的服务函数。
+ RPC层:服务客户端在该层生成用于调用服务函数的RPC代码与RPC数据。Service Server会根据传递过来的RPC代码查找相应的函数，并将RPC数据传递给查找到的函数。
+ IPC层:该层将RPC层生成的RPC代码与RPC数据封装成Binder IPC数据，以便它们传递给Binder Driver;
+ Binder Driver层:接收来自IPC层的Binder IPC数据，查找包含指定服务的Service Server，并将数据传递给查找到的Service Server.

3.为什么采用Binder机制实现IPC  

学过Linux的同学都知道，在Linux中，已经有管道，消息队列，共享内存，信号量，Socket等IPC通信机制,为什么Google选择了Binder，而不使用原有的IPC机制呢？  

主要是以下几方面的原因：  

1) Binder的传输效率和可操作性很好  
Socket由于传输效率太低，所以被排除。而消息队列和管道又采用存储-转发的方式,使用它们进行IPC通信时，需要经过2次内存拷贝，效率太低。  

为什么消息队列和管道的数据传输需要经过2次内存拷贝呢？首先，数据先从发送方的缓存区(即Linux中的用户存储空间)拷贝到内存开辟的缓存区(即Linux内核存储空间)中，是第1次拷贝。接着，再从内核缓存区拷贝到接收方的缓存区(也是Linux中的用户存储空间),这是第2次拷贝。  

而采用Binder机制的话，则只需要进行1次拷贝即可，因为从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的，因此只需要1次拷贝即可。  

而共享内存方式，虽然使用它进行IPC通信时进行的内存拷贝次数是0，但是由于操作复杂，故没有采用。  

2)Binder机制能够很好地实现Client-Server架构  
对于Android系统，Google想提供一套基于Client-Server的通信方式。  
例如，将MediaPlayer Service, Camera Service等不同的服务都由不同的Server提供，当Client需要获取某个Server的服务时，只需要Client向Server发送相应的请求，Server收到请求之后进行处理，处理完毕后再将反馈内容发送给Client.  

但是，目前Linux支持的管道，消息队列，共享内存，信号量，Socket等IPC手段中，只有Socket是Client-Server的通信方式 。但是Socket主要用于网络间通信以及本机中进程间的低速通信，它的传输效率太低，并不适合高速IPC通信。  

3)Binder机制的安全性高  

传统的Linux IPC机制没有任何安全措施，完全依赖上层协议来确保。传统IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份。而Binder机制则为每个进程分配了UID/PID来作为鉴别身份的标示，并且在Binder通信时会根据UID/PID进行有效性检测。  

4.UML图  

Binder机制涉及到代理类、Stub类，以及与Binder Driver对话的IPCThreadState类则，整体的UML图如下所示：  

![binder_uml](http://7xn1yt.com1.z0.glb.clouddn.com/binder_uml.png)

+ IBinder、BBinder、BpBinder:IBinder类是对Android Binder的抽象，它有BBinder、BpBinder两个子。其中BBinder类负责接收RPC代码与数据，并在Binder Driver内部生成Binder节点。BpBinder类保存着目标服务的Handle信息，用于Binder Driver查找Service Server的Binder节点的过程中;

+ IInterface、BnInterface、BpInterface:IInterface类提供类型变换功能，将服务或服务代理类转换为IBinder类型。实际的类型转换是由BnInterface、BpInterface两个类完成，BnInterface将服务类转换成IBinder类型，BpInterface则将服务代理类转换成IBinder类型。在通过Binder Driver传递Binder对象时，必须进行类型转换，比如在向系统注册服务时，需要先将服务类转换成IBinder,再将其转递给Context Manager.

+ ProcessState、IPCThreadState: ProcessState类用来管理Binder Driver,IPCThreadState类用来支持服务客户端、Service Server与Binder Driver间的Binder IPC通信;

+ Parcel: 在服务与服务代理间进行BinderIPC时，Parcel类负责保存Binder IPC数据。Parcel类可以处理的数据有C语言的基本数据结构、基本数据结构数组、Binder对象、文件描述符等。

5. Service Manager与Context Manager  

在Android系统中存在着各种各样的服务，如管理音频设备的Audio Flinger服务、管理帧缓冲设备的Surface Flinger服务，还有管理相机设备的Camera服务等。这些服务的信息与目录都是由Context Manager负责管理的。  

而Service Manager则相当于Context Manager的服务代理。需要注意的是,Service Manager分为管理Java系统服务的Service Manager和管理本地系统服务的Service Manager.下图很好地表示了它们的关系：  

![context_manager](http://7xn1yt.com1.z0.glb.clouddn.com/context_manager.png)

+ Java系统服务通过Java层的Service Manager注册服务，本地系统服务通过C/C++层的Service Manager注册服务 ;
+ Java层的Service Manager通过JNI与C/C++层的Service Manager连接在一起，C/C++层的Service Manager通过Binder Driver与Context Manager进行Binder RPC;

6.内核空间的重要结构体  

内核空间的Binder设计有3个非常重要的结构体:binder_proc, binder_node和binder_ref.在后面会有Blog专门分析它们，这里只是简要说一下它们的作用。 它们的作用如下：  

+ binder_proc是描述进程上下文信息的，每一个用户空间的进程都对应一个binder_proc结构体
+ binder_node是Binder实体对应的结构体，它是Server在Binder驱动中的体现
+ binder_ref是Binder引用对应的结构体，它是Client在Binder驱动中的体现

![binder_proc](http://7xn1yt.com1.z0.glb.clouddn.com/binder_struct.png)

1)Binder实体红黑树是保存binder_proc对应的进程所包含的Binder实体的，而Binder实体是与Server的服务对应的。可以将Binder实体红黑树理解为Server进程中包含的Server服务的红黑树;  
2)上图中有两棵Binder引用红黑树，这两棵树所包含的Binder引用都是一样的。不同的是，红黑树的排序基准不同，一个是以Binder实体来排序，而另一个则是以Binder引用描述(Binder引用描述实际上就是一个32位的整型数)来排序。以Binder引用描述的红黑树是为了方便进行快速查找。  

下图是binder_node结构体的示意图：

![binder_node](http://7xn1yt.com1.z0.glb.clouddn.com/binder_node.png)




