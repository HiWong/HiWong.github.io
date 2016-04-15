---
layout: post
title: "Android-Universal-Image-Loader完全解析及改进(02)"
date: 2015-10-16 08:10:35 +0800
comments: true
categories: Android
---
在 [上一篇博客](http://blog.imallen.wang/blog/2015/09/28/universalimageloaderwan-quan-jie-xi-ji-gai-jin/) 中讲了UIL的整体实现流程以及使用方法。本文将进行第2,3点的分析，即UIL整体架构分析和UIL重点类分析。

2.UIL整体架构分析

2.1 UIL工作流程

图片缓存库的设计思想 <!--more-->

![uil-flow](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_Flow.png)

UIL的实现流程

![UIL specfic flow](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_architite_02.png)

由上图可看出，UIL的重点在于分配任务的ImageLoaderEngine，相应的任务以及MemoryCache,DiskCache.

2.2 UIL整体架构

UIL的整体架构如下图所示

![UIL whole architect](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_Analysis04.png)

从上图可见，UIL主要分为以下模块：  

1)ImageLoader,ImageEngine;  
ImageLoader的主要工作是提供对外的接口，如init(...),displayImage(...),loadImage(...),loadImageSync(...),  
resume(),pause(),stop(),destroy();  
ImageEngine的主要作用是进行任务的分发与管理；

2)LoadAndDisplayImageTask,ProcessAndDisplayImageTask,DisplayBitmapTask;  
LoadAndDisplayImageTask:负责图片的加载及显示工作；  
ProcessAndDisplayImageTask:负责图片的处理及显示工作；  
DisplayBitmapTask:负责图片的显示工作；


3)MemoryCache,DiskCache,ImageDownloader,ImageDecoder;  
MemoryCache:内存缓存；  
DiskCache:磁盘缓存；  
ImageDownloader:图片下载器，注意它不只能处理HTTP和HTTPS这两种类型，还能处理CONTENT,ASSETS,DRAWABLE等类型；  
ImageDecoder:图片解码器，从字节流中进行解码从而获得Bitmap；  

2.3 UIL重点类分析  
2.3.1 重点类关系图  
![uil_uml](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_UML.png)

下面将详细分析上图中各类的作用以及相互关系。

2.3.2 ImageLoader,ImageEngine
1)ImageLoader:单例类，其主要作用是提供对外的接口，如init(...),displayImage(...),  
loadImage(...),loadImageSync(...),resume(),pause(),stop(),destroy();  
ImageLoader中最核心的方法是displayImage(String uri,ImageAware imageAware,DisplayImageOptions options,ImageLoadingListener listener,ImageLoadingProgressListener progressListener),各个形参的作用如下。  
uri:图片路径，可能为url也可能是assets或drawable等local resource;  
imageAware:用于包装View的接口；  
options:显示相关的设置，如加载前、加载中、加载失败的占位图片，图片是否需要在磁盘或内存缓存等；  
listener:图片加载各种时刻的回调接口，包括开始加载、加载失败、加载成功、取消加载时的回调函数；
progressListener:加载进度的回调接口；

下面是displayImage()的流程：

![displayImage_workflow](http://7xn1yt.com1.z0.glb.clouddn.com/displayImage_workflow.png)

2)ImageEngine:单例类，其主要作用是进行任务的分发与管理。其内部成员变量及作用如下。  
configuration:配置项，其中含有用于执行任务的线程池等相关参数；  

taskExecutor:当diskCache中不存在相应的图片文件时，利用taskExecutor从uri获取图片；  

taskExecutorForCachedImages:当diskCache中存在相应的图片文件是，利用taskExecutorForCachedImages从diskCache中获取图片；
从下面的log截图可以看出有diskCache和无diskCache时的线程在不同的线程池中，印证了上面的说法。

![thread_pool_name](http://7xn1yt.com1.z0.glb.clouddn.com/thread_pool_name.png)

其中的uil-pool-1是taskExecutor,uil-pool-2是taskExecutorForCachedImages;

**cacheKeysForImageAwares:由ImageAwared的id作key,memoryCacheKey作为value的map，注意它是线程安全的；这个成员非常重要，原因是它是判断ImageAware是否复用的指标，即如果当前任务中的memoryCacheKey和cacheKeysForImageAwares中ImageAware对应的memoryCacheKey不一致时，就可断定ImageAware已经被复用了，此时要立即取消当前任务，否则不仅线程池的负担加重，而且会显示错乱。**  

paused:它与pauseLock结合使用可控制暂停任务的执行；  

networkDenied:是否禁用网络获取图片;  

slowNetwork:是否为慢网络；  

ImageLoaderEngine最重要的两个方法是submit(LoadAndDisplayImageTask):void及submit(ProcessAndDisplayImageTask):void，分别用于执行加载、显示的任务以及处理、显示的任务。


2.3.3 LoadAndDisplayImageTask,ProcessAndDisplayImageTask,DisplayBitmapTask  
1)LoadAndDisplayImageTask  
当内存缓存中不存在相应的bitmap是，就要利用LoadAndDisplayImageTask从DiskCache,network或其他resource处获取bitmap并显示。  
以下是其主要成员变量的作用。  
engine:用于从engine中获取是否要暂停等信息；  

imageLoadingInfo:加载图片相关信息；  

handler:用于将加载结果传递到UI线程；  

configuration:获取全局配置信息；  

downloader:用于从network或本地资源中获取字节流;  

networkDeniedDownloader:禁用网络时用于从本地资源中获取字节流;  

slowNetworkDownloader:慢网络时用于从network获取字节流;  

decoder:解码器;  

options:图片显示相关信息;  

listener:图片加载监听器;  

progressListener:图片加载进度监听器;  


下面是其执行流程。  

![LoadAndDisplayImageTask_workflow](http://7xn1yt.com1.z0.glb.clouddn.com/LoadAndDisplayImageTask.png)

下面是tryLoadBitmap()的执行流程。  

![tryLoadBitmap](http://7xn1yt.com1.z0.glb.clouddn.com/try_load_bitmap.png)

2)ProcessAndDisplayImageTask  
它的作用就是如果有后处理的话，则先对图片进行后处理，然后进行显示。其主要成员变量如下。  

engine:当异步并且handler为空时，利用它的回调事件;  

bitmap:加载结果;  

imageLoadingInfo:其中含有图片后处理器;  

handler:用于将bitmap从工作线程传递到主线程;  

由于其流程很简单,此处不再给出流程图。  

3)DisplayBitmapTask  
用于通知监听器并进行图片的显示。其主要成语变量如下。  
bitmap:加载并进行后处理(如有)的结果;  

imageUri:图片路径;  

imageAware:View的包装对象;  

memoryCacheKey:内存缓存key;  

displayer:图片显示对象，其中包含动画等显示效果;  

listener:图片加载监听器;  

engine:用于取消imageAware的任务(若存在);  

loadedFrom:加载源;  

下面是其流程。  

![DisplayBitmapTask_workflow](http://7xn1yt.com1.z0.glb.clouddn.com/DisplayBitmapTask.png)

至此，ImageLoader中最核心的几个类已经分析完毕，为了避免篇幅过长，剩下的重点类分析将放在下一篇blog中给出。