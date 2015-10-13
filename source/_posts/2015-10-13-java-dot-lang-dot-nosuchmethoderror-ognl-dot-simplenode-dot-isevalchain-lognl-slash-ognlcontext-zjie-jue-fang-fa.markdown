---
layout: post
title: "java.lang.NoSuchMethodError: ognl.SimpleNode.isEvalChain(Lognl/OgnlContext;)之解决方法"
date: 2015-10-13 16:33:28 +0800
comments: true
categories: javaee
---

运行JavaEE项目时出现如下错误：  

java.lang.NoSuchMethodError: ognl.SimpleNode.isEvalChain(Lognl/OgnlContext;<!--more-->)Z

	at com.opensymphony.xwork2.ognl.OgnlUtil.isEvalExpression(OgnlUtil.java:245)
	at com.opensymphony.xwork2.ognl.OgnlUtil.checkEnableEvalExpression(OgnlUtil.java:281)
	at com.opensymphony.xwork2.ognl.OgnlUtil.compile(OgnlUtil.java:269)
	at com.opensymphony.xwork2.ognl.OgnlUtil.setValue(OgnlUtil.java:230)
	at com.opensymphony.xwork2.ognl.OgnlUtil.setValue(OgnlUtil.java:226)
	at com.opensymphony.xwork2.ognl.OgnlUtil.internalSetProperty(OgnlUtil.java:463)
	at com.opensymphony.xwork2.ognl.OgnlUtil.setProperties(OgnlUtil.java:118)
	at com.opensymphony.xwork2.ognl.OgnlUtil.setProperties(OgnlUtil.java:145)
	at com.opensymphony.xwork2.ognl.OgnlUtil.setProperties(OgnlUtil.java:132)
	at com.opensymphony.xwork2.ognl.OgnlReflectionProvider.setProperties(OgnlReflectionProvider.java:58)
	at com.opensymphony.xwork2.factory.DefaultInterceptorFactory.buildInterceptor(DefaultInterceptorFactory.java:43)
	at com.opensymphony.xwork2.ObjectFactory.buildInterceptor(ObjectFactory.java:202)
	at com.opensymphony.xwork2.config.providers.InterceptorBuilder.constructInterceptorReference(InterceptorBuilder.java:70)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.lookupInterceptorReference(XmlConfigurationProvider.java:1110)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.loadInterceptorStack(XmlConfigurationProvider.java:928)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.loadInterceptorStacks(XmlConfigurationProvider.java:941)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.loadInterceptors(XmlConfigurationProvider.java:964)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.addPackage(XmlConfigurationProvider.java:533)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.loadPackages(XmlConfigurationProvider.java:292)
	at org.apache.struts2.config.StrutsXmlConfigurationProvider.loadPackages(StrutsXmlConfigurationProvider.java:112)
	at com.opensymphony.xwork2.config.impl.DefaultConfiguration.reloadContainer(DefaultConfiguration.java:258)
	at com.opensymphony.xwork2.config.ConfigurationManager.getConfiguration(ConfigurationManager.java:67)
	at org.apache.struts2.dispatcher.Dispatcher.init_PreloadConfiguration(Dispatcher.java:445)
	at org.apache.struts2.dispatcher.Dispatcher.init(Dispatcher.java:489)
	at org.apache.struts2.dispatcher.ng.InitOperations.initDispatcher(InitOperations.java:74)
	at org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter.init(StrutsPrepareAndExecuteFilter.java:57)
	at org.apache.catalina.core.ApplicationFilterConfig.initFilter(ApplicationFilterConfig.java:279)
	at org.apache.catalina.core.ApplicationFilterConfig.getFilter(ApplicationFilterConfig.java:260)
	at org.apache.catalina.core.ApplicationFilterConfig.<init>(ApplicationFilterConfig.java:105)
	at org.apache.catalina.core.StandardContext.filterStart(StandardContext.java:4828)
	at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5508)
	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:150)
	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1575)
	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1565)
	at java.util.concurrent.FutureTask$Sync.innerRun(Unknown Source)
	at java.util.concurrent.FutureTask.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(Unknown Source)
	at java.lang.Thread.run(Unknown Source)

这其实是Struts版本与ognl版本不兼容导致的，自己所用的Struts为Struts2.3.16.3版本，要搭配的Ognl版本为Ognl3.0.6，而自己一开始搭配的是Ognl3.0，所以才会出现上述错误。修改之后就OK了。




