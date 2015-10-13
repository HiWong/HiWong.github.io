---
layout: post
title: "Hibernate4.3.6 Final+Spring3.0.5配置出错提示及解决方法"
date: 2015-10-13 16:56:15 +0800
comments: true
categories: javaee
---
	1. Caused by: org.hibernate.cache.NoCacheRegionFactoryAvailableException: Second-level cache is used in the application, but property hibernate.cache.region.factory_class is not given, please either disable second  level cache or set correct region factory class name to property hibernate.cache.region.factory_class (and make sure the second level cache provider, hibernate-infinispan, for example, is available in the classpath).

原因在于hibernate4.0在hibernate.cfg.xml配置二级缓存和hibernate3.3有所不同,本例子<!--more-->用的是 Hibernate-core.4.3.6.Final，实际上从4.0开始就不一样，二级缓存的配置对比如下所示： 


4.0以上配置如下：  

	<property name="hibernate.cache.use_second_level_cache">true</property>
	<property name="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</property>

3.3配置如下：  

	<property name="hibernate.cache.use_second_level_cache">true</property>
	<property name="cache.provider_class ">org.hibernate.cache.EhCacheProvider</property>

但是如果仅仅是修改上述配置的话，还是会出错，因为要使用二级缓存的话，还需要引用相应的jar包，即hibernate-release-4.3.6.Final\lib\optional\ehcache下的jar包也要拷贝到lib中，否则会出现Unable to create requested service [org.hibernate.cache.spi.RegionFactory]的错误。  

2.nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'attendTypeDao' is defined
这个错误提示很明显：没有定义相应的bean，仔细察看之后发现原来是自己太粗心，在定义bean时将"attendTypeDao"写成了"attentTypeDao"导致  

3.错误  

	!MESSAGE Server Tomcat v7.0 Server at localhost was unable to start within 45 seconds. If the server requires more time, try increasing the timeout in the server editor.

修改 workspace\.metadata\.plugins\org.eclipse.wst.server.core\servers.xml文件。

	<servers>
	<server hostname="localhost" id="JBoss v5.0 at localhost" name="JBoss v5.0 at
	localhost" runtime-id="JBoss v5.0" server-type="org.eclipse.jst.server.generic.jboss5"
	server-type-id="org.eclipse.jst.server.generic.jboss5" start-timeout="1000" stop-
	timeout="15" timestamp="0">
	<map jndiPort="1099" key="generic_server_instance_properties" port="8090"
	serverAddress="127.0.0.1" serverConfig="default"/>
	</server>
	</servers>

把 start-timeout="45" 改为 start-timeout="100" 或者更长.

就酱紫，终于部署成功了。但是实际部署时每个人遇到的问题可能都会有所不同，按它的出错提示信息仔细分析一般都可以找到原因


