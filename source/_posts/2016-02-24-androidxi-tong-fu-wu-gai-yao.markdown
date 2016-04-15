---
layout: post
title: "Android系统服务概要"
date: 2016-02-24 11:33:52 +0800
comments: true
categories: android_deep_analysis
---
1.Android系统服务  
Android系统服务提供系统最基本、最核心的功能，如设备控制、位置信息、通知设定、消息显示等。这些服务分别存在于Application Framework与Libraries层之中。如下图所示<!--more-->：  

![system_service](http://7xn1yt.com1.z0.glb.clouddn.com/system_service1.png)

2.系统服务分类  

系统服务分为本地系统服务和Java系统服务。  
2.1 本地系统服务  
本地系统服务使用C/C++编写，运行在Libraries层，主要包含Audio Flinger,Surface Flinger等。  

其中Audio Flinger服务混合多种Android应用程序的音频数据，并发送到耳机、扬声器等音频输出设备中，所有音频数据均经由Audio Flinger进行输出。  

Surface Flinger是Android Multimedia的一部分，在Android的实现中，它用于提供系统范围内的surface composer功能，能够将各种应用程序的Surface组合后渲染到Frame Buffer设备中。  

2.2 Java系统服务  

在Android启动时，Java系统服务由SystemServer系统进程启动，具体是如何启动的，可以查看博客[Zygote完全解析(2)](http://blog.imallen.wang/blog/2016/02/24/zygotewan-quan-jie-xi-2)。Java系统服务分为核心平台服务与硬件服务。  

2.2.1 核心平台服务(Core Platform Service)  

一般来说，核心平台服务不会直接与Android应用程序进行交互，但它们是Android Framework运行所必需的服务，其包含的主要服务为：  

+ Activity Manager Service:管理所有Activity的生命周期与堆栈
+ Window Manager Service:位于Surface Flinger之上，将要绘制到机器画面上的内容传递给Surface Flinger
+ Package Manager Service:加载apk文件的信息，提供信息显示系统中设置了哪些包，以及加载了哪些包

2.2.2 硬件服务(Hardware Service)  

该服务提供了一系列API，用于控制底层硬件，主要包含如下服务：  

+ Alarm Manager Service:在特定时间后运行指定的应用程序，就像定时器;
+ Connectivity Service:提供有关网络当前状态的信息
+ Location Service:提供终端当前的位置信息
+ Power Service:设备电源管理
+ Sensor Service:提供Android中各种传感器(磁力传感器、加速度传感器)的感应值
+ Telephony Service:提供话机状态及电话服务
+ Wifi Service:控制无线网络连接，如AP搜索、连接列表管理等 

2.2.3 如何使用Java系统服务  

无论是在Framework内部，还是Android应用程序中，若想使用Java系统服务，必须使用能够与各服务通信的Local Manager对象。  

比如，应用程序若想使用Location Service，获取终端设备当前的位置信息，需要先调用getSystemService()函数，创建与Location Service相应的Local Manager对象，而后应用程序使用生成的Local Manager对象，调用Location Service提供的各种函数，执行相应的功能。  

2.3 运行系统服务  

我们知道，在应用程序中，我们在使用一个服务前，需要先调用startService()函数以启动相应的服务。与之不同的是，使用系统服务时，客户端不需要先启动它，直接调用getSystemService()使用即可。因为在Android系统的启动过程中，init进程已经启动了这些系统服务。  

在Android启动时，系统服务具体由媒体服务器(Media Server)与系统服务器(System Server)两个系统进程运行。Media Server用来启动除Surface Flinger之外的Audio Flinger、MediaPlayer Service等本地服务。而System Server是Zygote最初生成的基于Java的进程，它会启动所有Java系统服务和一个本地系统服务：Surface Flinger.  

由于在博客[Zygote完全解析(2)](http://blog.imallen.wang/blog/2016/02/24/zygotewan-quan-jie-xi-2)中已经详细介绍过System Server加载Java系统服务和Surface Flinger的过程，所以此处不再赘述。  

至此，我们可以用下面这张图来归纳Android启动时生成系统服务的过程：  

![init_system_service](http://7xn1yt.com1.z0.glb.clouddn.com/init_system_service.png)

2.3.1 Media Server的运行过程  

Media Server是个系统进程，它运行Audio Flinger、Media Player Service、Camera Service、Audio Policy Service等本地系统服务。它由init进程启动运行，在init.rc脚本文件中，可以看到相关脚本，如下所示：  
``` sh init.rc
service media /system/bin/mediaserver
user media
group system audio camera graphics inet net_bt net_bt_admin
```
下面是frameworks/base/media/mediaserver/main_mediaserver.cpp中的main()函数代码：  

``` cpp main_mediaserver.cpp
int main(int argc,char** argv)
{
   sp<ProcessState>proc(ProcessState::self());
   sp<IServiceManager>sm=defaultServiceManager();
   LOGI("ServiceManager:%p",sm.get());
   AudioFlinger::instantiate();
   MediaPlayerService::instantiate();
   CameraService::instantiate();
   AudioPolicyService::instantiate();
   ProcessState::self()->startThreadPool();
   IPCThreadState::self()->joinThreadPool();
}
```
要完成理解代码需要等我们学习完Binder机制之后，简单地说就是初始化并注册了AudioFlinger,MediaPlayerService,CameraService等。比如AudioFlinger::instantiate()的代码如下：  

``` cpp AudioFlinger.cpp
void AudioFlinger::instantiate(){
    defaultServiceManager()->addService(String16("media.audio_flinger"),new AudioFlinger());
}
```
其中defaultServiceManager()返回的是BpServiceManager对象，addService()即为注册函数。详细的分析将在后面Binder机制的分析中给出。
