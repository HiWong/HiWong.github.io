---
layout: post
title: "Struts2中指定的校验文件不起作用的原因"
date: 2015-10-13 17:16:49 +0800
comments: true
categories: javaee
---
我们知道，如果要为某个Action指定校验文件，那么就要将"Action名-validation.xml"的校验文件放在与该Action在同一路径下，并且在struts.xml中指定<result name="input>的值，在input对应的文件中加入<s:fielderror/>即可。此时如果输入不符合校验规则，就不会跳转到相应的action，而是仍然跳转回input文件，并显示相应的校验提示<!--more-->。  

但是自己在指定了校验文件之后，却发现始终不起作用，后来终于发现问题，原来是自己贪图方便，直接从别处复制过来校验配置文件的dtd信息，但是这个复制过来的dtd信息跟自己现在这个版本的Struts所用的dtd信息并不相同，从而导致校验不起作用。  

解决方法很简单，就是在lib中找到自己所用的xwork-core文件，比如我的是xwork-core-2.3.16.3.jar，用解压工具查看其中的dtd文件，一般有多个，我查看的是xwork-validator-1.0.3.dtd,里面内容如下：  

	<?xml version="1.0" encoding="UTF-8"?>

	<!--
	  XWork Validators DTD.
	  Used the following DOCTYPE.

	  <!DOCTYPE validators PUBLIC
	  		"-//Apache Struts//XWork Validator 1.0.3//EN"
	  		"http://struts.apache.org/dtds/xwork-validator-1.0.3.dtd">
	-->


	<!ELEMENT validators (field|validator)+>

	<!ELEMENT field (field-validator+)>
	<!ATTLIST field
		name CDATA #REQUIRED
	>

	<!ELEMENT field-validator (param*, message)>
	<!ATTLIST field-validator
		type CDATA #REQUIRED
	    short-circuit (true|false) "false"
	>

	<!ELEMENT validator (param*, message)>
	<!ATTLIST validator
		type CDATA #REQUIRED
	    short-circuit (true|false) "false"
	>

	<!ELEMENT param (#PCDATA)>
	<!ATTLIST param
	    name CDATA #REQUIRED
	>

	<!ELEMENT message (#PCDATA|param)*>
	<!ATTLIST message
	    key CDATA #IMPLIED
	>

只要将下面的片段复制到校验文件中即可：  

	<pre name="code" class="html"><!DOCTYPE validators PUBLIC
	  		"-//Apache Struts//XWork Validator 1.0.3//EN"
	  		"http://struts.apache.org/dtds/xwork-validator-1.0.3.dtd">
	-->

后面尝试了一下，发现用xwork-validator-1.0.2.dtd中的配置信息也可以，这应该只是版本的问题，但是一定要是自己的xwork-core支持的版本才行。