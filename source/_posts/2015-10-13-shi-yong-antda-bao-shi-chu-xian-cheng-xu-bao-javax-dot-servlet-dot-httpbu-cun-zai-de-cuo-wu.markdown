---
layout: post
title: "使用Ant打包时出现程序包javax.servlet.http不存在的错误"
date: 2015-10-13 16:26:44 +0800
comments: true
categories: javaee
---

显然，出现这个错误的原因是缺少相应的jar包。具体原因由于servlet和JSP不是Java平台JavaSE（标准版）的一部分，而是Java EE（企业版）的一部分，因此，必须告知编译器servlet的位置.  

解决方法有以下三种：  

第一，将apache-tomcat-7.0.55\lib\servlet-api.jar添加到环境变量中。  

第二，直接将servlet-api.jar添加到lib文件夹中。  

第三，在build.xml中增加相应的fileset，将对应的servlet-api.
jar包含进去，其实就是为单个项目的classpath增加jar包的路径，原理上和第一种方法是一样的。  

