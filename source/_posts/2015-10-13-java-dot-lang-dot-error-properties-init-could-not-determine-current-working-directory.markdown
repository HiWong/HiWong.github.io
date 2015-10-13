---
layout: post
title: "java.lang.Error:Properties init:Could not determine current working directory"
date: 2015-10-13 13:15:41 +0800
comments: true
categories: java
---
今天在CentOS中安装好jdk并设置环境变量之后，输入java -version竟然出现以下错误：  

	java.lang.Error:Properties init: Could not determine current working directory

    at java.lang.System.initProperties(Native Method)

    at java.lang.System.initializeSystemClass(System.java:1069)

其实并不是版本不兼容或者是环境变量设置错误的问题，真正的原因是自己之前在某个文件夹中打开terminal，但是后面在另一个termianl中把这个文件夹给删除了，但是自己却没有cd到别的路径下，从而导致出现无法确定当前目录的情况出现。后cd到另外一个真实存在的目录即可。