---
layout: post
title: "Android编译时出现'Unable to execute dex: Multiple dex files define Lcom...'"
date: 2015-10-13 19:49:01 +0800
comments: true
categories: android
---
中午从SVN Update项目代码之后，对Project进行Refresh之后进行编译，却总是提示"Unable to execute dex: Multiple dex files define Lcom/nostra13/universalimageloader/cache/disc/Disc"，看代码并没有错误，于是又Clean up了一次，还是没用。重启依然无效，后面查找资料发现原来是有重复的jar文件被引用，于是查看libs下的文件，发现有两个ImagerLoader(一个是ImageLoader_383751.jar,另一个是ImageLoader_544751)，将前者删除后再编译，果然不再报错。