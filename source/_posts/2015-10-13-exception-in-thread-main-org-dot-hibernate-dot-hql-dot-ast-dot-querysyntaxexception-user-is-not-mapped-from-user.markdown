---
layout: post
title: " Exception in thread main org.hibernate.hql.ast.QuerySyntaxException: User is not mapped [from User"
date: 2015-10-13 13:11:47 +0800
comments: true
categories: javaee
---
	Exception in thread "main" org.hibernate.hql.ast.QuerySyntaxException: User is not mapped [from User user where user.name=?0 and user.pass=?1]
	at org.hibernate.hql.ast.util.SessionFactoryHelper.requireClassPersister(SessionFactoryHelper.java:180)
	at org.hibernate.hql.ast.tree.FromElementFactory.addFromElement(FromElementFactory.java:111)
	at org.hibernate.hql.ast.tree.FromClause.addFromElement(FromClause.java:93)
	at org.hibernate.hql.ast.HqlSqlWalker.createFromElement(HqlSqlWalker.java:322)
	at org.hibernate.hql.antlr.HqlSqlBaseWalker.fromElement(HqlSqlBaseWalker.java:3441)
	at org.hibernate.hql.antlr.HqlSqlBaseWalker.fromElementList(HqlSqlBaseWalker.java:3325)
	at org.hibernate.hql.antlr.HqlSqlBaseWalker.fromClause(HqlSqlBaseWalker.java:733)
	at org.hibernate.hql.antlr.HqlSqlBaseWalker.query(HqlSqlBaseWalker.java:584)
	at org.hibernate.hql.antlr.HqlSqlBaseWalker.selectStatement(HqlSqlBaseWalker.java:301)
	at org.hibernate.hql.antlr.HqlSqlBaseWalker.statement(HqlSqlBaseWalker.java:244)
	at org.hibernate.hql.ast.QueryTranslatorImpl.analyze(QueryTranslatorImpl.java:254)
	at org.hibernate.hql.ast.QueryTranslatorImpl.doCompile(QueryTranslatorImpl.java:185)
	at org.hibernate.hql.ast.QueryTranslatorImpl.compile(QueryTranslatorImpl.java:136)
	at org.hibernate.engine.query.HQLQueryPlan.<init>(HQLQueryPlan.java:101)
	at org.hibernate.engine.query.HQLQueryPlan.<init>(HQLQueryPlan.java:80)
	at org.hibernate.engine.query.QueryPlanCache.getHQLQueryPlan(QueryPlanCache.java:124)
	at org.hibernate.impl.AbstractSessionImpl.getHQLQueryPlan(AbstractSessionImpl.java:156)
	at org.hibernate.impl.AbstractSessionImpl.createQuery(AbstractSessionImpl.java:135)
	at org.hibernate.impl.SessionImpl.createQuery(SessionImpl.java:1770)
	at com.login.hibernate.HqlQuery.findUser(HqlQuery.java:17)
	at com.login.hibernate.HqlQuery.main(HqlQuery.java:27)

其实错误提示说得很清楚了，就是没有添加映射，只要在src下的hibernate.cfg.xml中增加相应类的映射即可
	