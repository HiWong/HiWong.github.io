---
layout: post
title: "Android-Universal-Image-Loader完全解析及改进(02)"
date: 2015-09-30 08:10:35 +0800
comments: true
categories: 
---
在 [上一篇博客](http://blog.imallen.wang/blog/2015/09/28/universalimageloaderwan-quan-jie-xi-ji-gai-jin/) 中讲了UIL的整体实现流程以及使用方法。本文将进行第2,3点的分析，即UIL整体架构分析和UIL重点类分析。

2.UIL整体架构分析

2.1 UIL工作流程

图片缓存库的设计思想

![uil-flow](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_Flow.png)

UIL的实现流程

![UIL specfic flow](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_architite_02.png)

由上图可看出，UIL的重点在于分配任务的ImageLoaderEngine，相应的任务以及MemoryCache,DiskCache.

2.2 UIL整体架构

UIL的整体架构如下图所示

![UIL whole architect](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_Analysis03.png)


