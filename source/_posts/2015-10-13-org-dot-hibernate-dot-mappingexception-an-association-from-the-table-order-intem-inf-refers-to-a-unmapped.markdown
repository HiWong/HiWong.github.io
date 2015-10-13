---
layout: post
title: "org.hibernate.MappingException: An association from the table order_intem_inf refers to a unmapped"
date: 2015-10-13 17:02:56 +0800
comments: true
categories: javaee
---
运行一个HIbernate的示例时出现如下错误信息   

	Exception in thread "main" java.lang.ExceptionInInitializerError
	at com.hibernate.utils.HibernateUtil.<clinit>(HibernateUtil.java:21)
	at org.hibernate.samples.PersonManager.main(PersonManager.java:23)
	Caused by: org.hibernate.MappingException: An association from the table order_intem_inf refers to an unmapped class: Product
	at org.hibernate.cfg.Configuration.secondPassCompileForeignKeys(Configuration.java:1805)
	at org.hibernate.cfg.Configuration.originalSecondPassCompile(Configuration.java:1739)
	at org.hibernate.cfg.Configuration.secondPassCompile(Configuration.java:1424)
	at org.hibernate.cfg.Configuration.buildSessionFactory(Configuration.java:1844)
	at org.hibernate.cfg.Configuration.buildSessionFactory(Configuration.java:1928)
	at com.hibernate.utils.HibernateUtil.<clinit>(HibernateUtil.java:16)
	... 1 more

从出错信息可知类Product没有被映射到，但是明明Product.hbm.xml文件存在，并且mapping信息也填写了啊，后来检查发现是在另一个引用到它的地方只写"Product"而不是完整的路径名。




