---
layout: post
title: "Java中String的比较"
date: 2015-10-13 19:44:17 +0800
comments: true
categories: java
---
说到String的比较，就不能不提到其常量池的问题，目前网上有许多解释，也有很多所谓的实例，但是有很多并非作者原创，而是转载而来，关键是其中许多实例并未经过作者的验证，有些我验证之后发现与其结果不符，说明其解释也有问题。我不敢保证我的理解是最贴近Java本质的，但是至少我可以保证我所有的代码和结果都是经过自己调试并运行过的，也正是如此我所有的运行结果都采用截图的形式<!--more-->，以保证内容真实。  

废话少说，先看下面一段代码：  

	import java.util.*;

	public class TestString
	{
	  public static void main(String[]args)
	  {
	    String s1="java";
	    String s2="java";
	    String s3=new String("java");
	    System.out.println(s1==s2);
	    System.out.println(s1==s3);


	    String str1="study";
	    String str2="java";
	    String str3=str1+str2;
	    System.out.println(str3=="studyjava");
	    
	    String str4="study"+"java";
	    System.out.println(str4=="studyjava");
	   
	  }
	}

输出结果如下图所示：   
![java_str_cmp01](http://7xn1yt.com1.z0.glb.clouddn.com/java_string_cmp01.png)

原因如下：  

1. 虽然s1与s2是两个引用，但是由于系统给String维护着一个常量池，它保证了每个字符串常量只有一个备份，因而s1和s2这两个字符串引用指向的是常量池中的同一个对象。  

2. 那为什么s1==s3的结果是false呢，这是因为s3指向的对象并不在常量池中，而是在堆中。  

3. 而至于str3=="studyjava"的结果为false,则是由于str1和str2并非直接的字符串常量，而是字符串引用，因而str3的值不能在编译时就确定，需要到运行时才能确定；所以如果改成str3="study"+str2;那么同样的str3=="studyjava"的值还是false，因为有一个是字符串引用，JVM没法在编译时就将其优化为指向常量池中的对象。  

但是对于两个字符串常量的连接(即使用"+")，JVM会在编译时就将其优化为连接后的值，即str4指向的就是常量池中的"studyjava"，因而str4=="studyjava"的值为true.  
