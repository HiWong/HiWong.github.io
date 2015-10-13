---
layout: post
title: "Unable to load configuration. - bean - jar:file:../../ComputerScience/JavaEE/workspace/.metadata解决办法"
date: 2015-10-13 16:44:02 +0800
comments: true
categories: javaee
---

在运行一个JavaEE的项目时，出现了如下的错误<!--more-->：  

	 unable to load configuration. - bean - jar:file:/../workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/wtpwebapps/WorkSystem/WEB-INF/lib/struts2-core-2.3.16.3.jar!/struts-default.xml:64:179
	at org.apache.struts2.dispatcher.Dispatcher.init(Dispatcher.java:501)
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
	Caused by: Unable to load configuration. - bean - jar:file:/../workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/wtpwebapps/WorkSystem/WEB-INF/lib/struts2-core-2.3.16.3.jar!/struts-default.xml:64:179
	at com.opensymphony.xwork2.config.ConfigurationManager.getConfiguration(ConfigurationManager.java:70)
	at org.apache.struts2.dispatcher.Dispatcher.init_PreloadConfiguration(Dispatcher.java:445)
	at org.apache.struts2.dispatcher.Dispatcher.init(Dispatcher.java:489)
	... 15 more
	Caused by: Unable to load bean: type:org.apache.struts2.dispatcher.multipart.MultiPartRequest class:org.apache.struts2.dispatcher.multipart.JakartaMultiPartRequest - bean - jar:file:/../workspace/.metadata/.plugins/org.eclipse.wst.server.core/tmp0/wtpwebapps/WorkSystem/WEB-INF/lib/struts2-core-2.3.16.3.jar!/struts-default.xml:64:179
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.register(XmlConfigurationProvider.java:245)
	at org.apache.struts2.config.StrutsXmlConfigurationProvider.register(StrutsXmlConfigurationProvider.java:102)
	at com.opensymphony.xwork2.config.impl.DefaultConfiguration.reloadContainer(DefaultConfiguration.java:234)
	at com.opensymphony.xwork2.config.ConfigurationManager.getConfiguration(ConfigurationManager.java:67)
	... 17 more
	Caused by: java.lang.NoClassDefFoundError: org/apache/commons/fileupload/RequestContext
	at java.lang.Class.getDeclaredConstructors0(Native Method)
	at java.lang.Class.privateGetDeclaredConstructors(Unknown Source)
	at java.lang.Class.getDeclaredConstructors(Unknown Source)
	at com.opensymphony.xwork2.config.providers.XmlConfigurationProvider.register(XmlConfigurationProvider.java:235)
	... 20 more
	Caused by: java.lang.ClassNotFoundException: org.apache.commons.fileupload.RequestContext
	at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1720)
	at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1571)
	... 24 more

出现这种情况的原因一般只有两个：第一，xwork的版本号太低；第二，未将所有的struts2必需的类库添加进去，一般来说使用struts2需要添加commons-fileupload-x.x.x.jar,commons-io-x.x.x.jar，freemarker-x.x.x.jar,javassist-x.x.ga.jar,struts2-core.x.x.x.jar和xwork-core-x.x.x.jar，其中x.x.x为相应的版本号。  

检查了一下，发现自己的struts2-core.2.3.16.3.jar与xwork-core-2.3.16.3.jar兼容，所以是缺少jar包的原因，仔细检查后发现是缺少了   commons-fileupload-x.x.x.jar,commons-io-x.x.x.jar，freemarker-x.x.x.jar,javassist-x.x.ga.jar这几个类库。  

