---
layout: post
title: "There is no Action mapped for action name mylogin!regist错误原因"
date: 2015-10-13 17:22:15 +0800
comments: true
categories: 
---
大家都知道，可以利用DMI(Dynamic Method Invocation，动态方法调用)进行一个Action对应多个表单动作，今晚尝试了登录与注册两个表单动作的示例时，却总是弹出"There is no Action mapped for action name mylogin!regist"的错误，check了好多遍，struts.xml及JSP文件都没有错，src也能编译通过，但是点击注册时却总是弹出上面<!--more-->的错误。  

折腾了半个多小时，终于找到原因，原来是没有打开配置项中的动态方法调用，因为Struts2默认是不允许动态方法调用的。找到了原因就好办了，有两种解决方法：  

第一种是在struts.xml中加上下面这句：

	<constant name="struts.enable.DynamicMethodInvocation" value="true"/>

第二种是在src下面新建一个properties文件，加上  

	struts.enable.DynamicMethodInvocation = true

这句。  

个人推荐使用第一种方法，原因是第二种方法其实是为了保证向下兼容才保留的，但是新的配置方法是使用xml文件进行配置。

