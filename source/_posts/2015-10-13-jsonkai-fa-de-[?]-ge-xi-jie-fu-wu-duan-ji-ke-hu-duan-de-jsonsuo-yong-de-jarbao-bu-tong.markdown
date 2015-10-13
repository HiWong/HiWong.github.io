---
layout: post
title: "json开发的一个细节:服务端及客户端的json所用的jar包不同"
date: 2015-10-13 12:24:39 +0800
comments: true
categories: javaEE
---
之前自己为了简便，没有从json的官网下载相关的jar包，而是自作聪明地把Android中的jar包copy过来直接用在服务端。但实际上Android sdk中json相关的jar包其实是json所有jar包的一个子集，即不完全，实际上它主要偏重解析而不是创建，因而使用它来作为服务端的jar包的话就会出现一些问题，特别是传递JavaBean的Array或者List时，这个问题纠结了很久，才发现是这个原因导致的。  

实际上服务端如果要使用json的<!--more-->话，需要引入以下jar包：  

    ezmorph-1.0.6.jar

    json-lib-2.4-jdk15.jar

    commons-beanutils-1.8.3.jar

    commons-collections-3.2.1.jar

    commons-lang-2.6.jar

    commons-logging-1.1.1.jar

下载地址为：[json相关jar包合集](http://download.csdn.net/detail/bettarwang/8459769)   

当然，大家也可以从json官网下载最新的jar包：http://commons.apache.org/index.html   




