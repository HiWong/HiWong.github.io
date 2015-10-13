---
layout: post
title: "畅销书对Java中Iterator的理解误区"
date: 2015-10-13 17:44:04 +0800
comments: true
categories: java
---
最近放假，闲来无事，便翻看以前看过的一些书，竟然发现有些书本（甚至是一些畅销书）对Java中Iterator有很大的误解，比如某畅销书在Collection那一章有这么一句话：“当使用Iterator对集合元素进行迭代时，Iterator并不是把集合元素本身传给了迭代变量，而是把集合元素的值传给了迭代变量，所以修改迭代变量的值对集合元素本身没有任何影响。”  

我们先看它举的例子，代码如下<!--more-->（注：原文中字符串为中文，此处按个人偏好将其翻译成了英文）：  

	import java.util.*;

	public class IteratorTest
	{
	  public static void main(String[]args)
	  {
	     Collection books=new HashSet();
	     books.add("Light Java EE Practice");
	     books.add("Crazy Java Textbook");
	     books.add("Crazy Android Textbook");
	     
	     Iterator it=books.iterator();
	     while(it.hasNext())
	     {
	        String book=(String)it.next();
	        System.out.println(book);
	        if(book.equals("Crazy Java Textbook"))
	        {
	           it.remove();
	        }
	        book="Test String";
	     }
	     System.out.println(books);
	   }
	}

其输出结果如下：  

![java_iterator01](http://7xn1yt.com1.z0.glb.clouddn.com/java_iterator01.png)

从这个输出结果来看：好像作者的结论挺对的。但是，其实这个结论很容易被证伪。如下面的一个例子：  

	import java.util.*;

	class Apple
	{
	  int weight;
	  public Apple(int weight)
	  {
	    this.weight=weight;
	   }
	   public String toString()
	   {
	      return "weight:"+weight;
	    }
	}

	public class IteratorSample
	{
	  public static void main(String[]args)
	  {
	     Collection collection=new HashSet();
	     Apple[]appleArray=new Apple[10];
	     for(int i=0;i<10;++i)
	     {
	        appleArray[i]=new Apple(i+200);
	        collection.add(appleArray[i]);
	     }
	     System.out.println(collection);
	     
	     Iterator iterator=collection.iterator();
	     Apple apple=(Apple)iterator.next();
	     apple.weight=300;
	     System.out.println(collection);
	   }
	}

输出结果如下：  

![java_iterator02](http://7xn1yt.com1.z0.glb.clouddn.com/java_iterator02.png)

显然，collection中的第一个元素被改变了。  

那么到底为什么会出现这种看起来截然相反的两种结果呢？  

问题就出在String上!在我的博客深刻理解Java中final的作用(一):从final的作用剖析String被设计成不可变类的深层原因一文中就讲过String是Java中一个很特殊的类，它的特殊之外就在于它是一个不可变类，即String对象一旦生成便不可改变，经常看到的对String变量的赋值其实是让引用指向新的String对象!  

也就是说，在本文中第一个例子中，语句String book=(String)it.next();的作用只是让book也指向it.next()所指向的对象，后面book="Test String"；只是让book这个引用指向一个新的字符串对象，显然这种改变不会也不可能影响原对象，it.next()所指向的另然是原来的那个对象，所以从结果来看好像作者的结论挺有道理。但是实际上Iterator.next()确实是提供了原对象的引用，对于除不可变类之外的其他对象，都可以通过这个引用来改变它（当然，也要有相应的接口）。本文中第二个例子就充分地说明了这一点，由于Apple并不是一个不可变类，故可通过iterator.next()来改变它指向的对象。  

可能有人会质疑，为什么第一个例子中就不是让it.next()这个引用指向新的对象呢？原因很简单：book和it.next()是两个独立的引用，而程序中只对book进行了赋值，并没有对it.next()进行赋值。看下面一个简单的例子即可知：  

	import java.util.*;

	public class StringTest
	{
	  public static void main(String[]args)
	  {
	     String str1="I miss you";
	     String str2=str1;
	     System.out.println("str1:"+str1);
	     System.out.println("str2:"+str2);
	     str2="Will you miss me?";
	     System.out.println("str1:"+str1);
	     System.out.println("str2:"+str2);
	  }
	}

输出结果如下：  

![java_iterator03](http://7xn1yt.com1.z0.glb.clouddn.com/java_iterator03.png)

显然，对str2的赋值根本就不影响str1，本文中第一个例子与其类似。  

以前看书总是全盘接收，很少持怀疑态度，现在才发现“尽信书不如无书”，即使是畅销书，特别是国内的一些书籍，结论未必严谨。带着批判的角度去看书，多一些独立思考，方能有更大的收获。  





