---
layout: post
title: "深刻理解Java中的String、StringBuffer和StringBuilder的区别"
date: 2015-10-13 19:33:22 +0800
comments: true
categories: java
---
首先简单地来梳理一下Java中String、StringBuffer和StringBuilder各自的含义。  

1.String类  

首先，它是线程安全的，即可以用于多线程编程中；  

其次，String类的对象是不可变的，即在定义时就确定了，类似String str="Hello";str+="Java";的语句其实是生成了新的对象，只是我们未察觉到而已。但是注意在大量的字符串新建对象时消耗就很可观，这时必须考虑采用StringBuffer或StringBuilder，否则会极大地降低程序的效率<!--more-->。  

2.StringBuffer类:  

首先，它也是线程安全的。  

其次，它是可变类，对它指向的字符串的操作都不会产生新的对象。  
每个StringBuffer对象都有一定的缓冲区容量，当字符串大小没有超过容量时，不会分配新的容量，当字符串大小超过容量时，会自动增加容量。因而它的效率要比String高。   

StringBuffer 类最常用的两个方法是 append 和 insert 方法，StringBuffer已经重载了这些方法，以接受任意类型的数据，所以小伙伴们，假设strBuffer是StringBuffer的对象，那么像strBuffer.append(3.14f);strBuffer.append(true);这样的语句都是完全合法的。  

 3.StringBuilder类：  

首先，它不是线程安全的，即只能用于单线程编程中。  

其次，它跟StringBuffer类似，即其对象也是一个可变的字符序列。但是要注意的是它下面几种构造方法：  

StringBuilder():创建一个容量为16的StringBuilder对象；  

StringBuilder(int capacity):创建一个容量为capacity的StringBuilder对象；  

StringBuilder(String s):创建一个包含s的StringBuilder对象，同时末尾添加16个空元素。  

StringBuilder(CharSequence cs):创建一个包含cs的StringBuilder对象，末尾附加16个空元素；  

综上可知，在线程同步方面，String和StringBuffer是线程安全的，而StringBuilder不是线程安全的；在执行效率上，StringBuilder>StringBuffer>String，因而在需要大量的进行字符串操作的单线程场合，应该昼使用StringBuilder以提高效率，在大量进行字符串操作的多线程情形，StringBuffer无疑是最佳的选择；而对于少量的字符串操作的单线程或多线程情形下，使用String则更为简单、方便。  

上面对这三个类进行了一下梳理，但这只是基础知识，根本谈不上深刻理解。取这么一个题目不是想哗众取宠，下面就开始结合我自己的一些经历谈一下自己的理解。  

我们都知道，String类的对象是不可变的，但是又考虑到Java中”一切皆引用“，以为在函数中传String引用可以实现值的改变，因而常常犯下面这样的错误：  

	import java.util.*;
	import java.io.*;

	public class StringTest
	{
	   private static void treatString(String str)
	   {
	     if(str.contains("hello"))
	     {
	       str="hello java";
	     }
	     else
	     {
	       str="enjoy java";
	     }
	   }

	   public static void main(String[]args)
	   {
	      String str1="hello world";
	      String str2="study java";
	      treatString(str1);
	      treatString(str2);
	      
	      System.out.println(str1);
	      System.out.println(str2);
	   }
	}
   
输出结果如下图所示：  

![java_string01](http://7xn1yt.com1.z0.glb.clouddn.com/java_string01.png)

本来期望传递引用从而改变字符串str1和str2的值，但是从输出结果看字符串的值却是根本没有被改变，这是为什么呢？  
原来确实传递的也是引用，但是与一般的对象不同，treatString(String)函数并没有对str指向的对象进行修改（或者说并没有在str指向的内存地址上进行修改），而是新建了一个String对象，但是这个新建的对象却是只有形参指向它，实参并没有指向它，实际上整个过程中实参str都始终还是指向最开始那个对象。所以不难理解为什么会有这样的输出结果。  

那么如果想要获得一般对象传递引用的效果该怎么办？  

很简单，第一种方法，也是我个人比较推荐的方法，就是返回值String而不是void，即将上面的代码修改为：  
	import java.util.*;
	import java.io.*;

	public class StringTest
	{
	   //private static void treatString(String str)
	    private static String treatString(String str)
	   {
	     if(str.contains("hello"))
	     {
	       str="hello java";
	     }
	     else
	     {
	       str="enjoy java";
	     }
	     
	     return str;
	   }

	   public static void main(String[]args)
	   {
	      String str1="hello world";
	      String str2="study java";
	      str1=treatString(str1);
	      str2=treatString(str2);
	      
	      System.out.println(str1);
	      System.out.println(str2);
	   }
	}
   
此时输出结果如下图所示：  

![java_string02](http://7xn1yt.com1.z0.glb.clouddn.com/java_string02.png)

显然，此时已经达到了我们预期的目的。  

到了这里，我再结合自己的另一个经历，就是一开始对String.replace(oldChar,newChar);这个函数不熟悉，以为只要调用它就可以实现str的改变(即以为直接str.replace(oldChar,newChar);就能实现str的改变），实际上要利用它的返回值才能改变str，即str=str.replace(oldChar,newChar);才是正确的做法。  

第二种方法，当然就是利用StringBuffer和StringBuilder喽，因为它们是对象可变的嘛。具体例子到后面再追加，今天先写到这里吧。晚安啦，小伙伴们！  