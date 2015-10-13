---
layout: post
title: " MySQL server version for the right syntax to use near 'type=InnoDB' at line 1"
date: 2015-10-13 17:06:55 +0800
comments: true
categories: javaee
---
在运行一个Hibernate的示例时，配置了<property name="hibernate.hbm2ddl.auto">update</property>属性，但是自动建表却一直不成功，出错信息为<!--more-->：  

    ERROR: HHH000388: Unsuccessful: create table info_table (id integer not null auto_increment, title varchar(255), content varchar(255), primary key (id)) type=InnoDB

    ERROR: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'type=InnoDB' at line 1

很显然，这是MySQL的版本问题导致的，实际上，在MySQL5.0以前，type=InnoDB是有效的SQL语句，但是自己用的是MySQL5.5版本，type=InnoDB不再有效了。  

解决办法就是修改hibernate.cfg.xml中的dialect属性，将  

	<property name="dialect">org.hibernate.dialect.MySQLInnoDBDialect</property>

修改为  

   <property name="hibernate.dialect">org.hibernate.dialect.MySQL5InnoDBDialect</property>

再次运行，发现可以自动建表了。