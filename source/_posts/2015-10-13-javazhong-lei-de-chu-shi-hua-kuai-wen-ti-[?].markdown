---
layout: post
title: "Java中类的初始化块问题(一)"
date: 2015-10-13 20:27:40 +0800
comments: true
categories: java
---
Java中类进行初始化的地方有两个，一个是构造函数，另一个就是初始化块。而初始化块分为静态初始化块和非静态初始化块，它们都在构造函数之前执行，但是二者略有不同，静态初始化块是在类被加载到内存就开始执行，而非静态初始化块是在创建对象时，构造之前被执行。由以下两个例子即可看出<!--more-->。  

例一：代码如下所示，输出结果为“static initial block"，如图1所示。这说明只加载了静态初始化块，因为此时只有类被加载到内存，但是并没有创建对象。  

	import java.util.*;

	public class Base
	{
	  private String name="Tom";
	  private int age=48;
	  
	  //静态初始化块
	  static
	  {
	    System.out.println("static initial block");
	   }
	 
	   //非静态初始化块
	  {
	    System.out.println("name="+name+" age="+String.valueOf(age));
	   }

	   public Base()
	   {
	      System.out.println("Base Constructor");
	   }
	   
	   public static void main(String[]args)
	   {
	     Base b;
	     //b=new Base();
	    }
	}

![java_init_block04](http://7xn1yt.com1.z0.glb.clouddn.com/java_init_block04.png)

例二：依然使用上面的代码，唯一的变化是将注释的b=new Base();的注释符去掉，即此时会新建对象，此时输出为：  

	static initial block

	name=Tom age=48

	Base Constructor

如图2所示。这说明静态初始化块在非静态初始化块之前执行，而非静态初始化块在构造函数之前执行  

![java_init_block05](http://7xn1yt.com1.z0.glb.clouddn.com/java_init_block05.png)

由于初始化块的执行顺序是由它们在类中出现的顺序决定的，即先出现的比后出现的要先执行，因而或许有人会质疑：如果将上面的代码中的非静态初始化块放在静态初始化块中，那非静态初始化块是不是就在静态初始化块之前执行呢？验证一下便知，将代码修改为如下：  

	import java.util.*;

	public class Base
	{
	  private String name="Tom";
	  private int age=48;
	  
	  {
	    System.out.println("name="+name+" age="+String.valueOf(age));
	   }

	  static
	  {
	    System.out.println("static initial block");
	   }


	   public Base()
	   {
	      System.out.println("Base Constructor");
	   }
	   
	   public static void main(String[]args)
	   {
	     Base b;
	     b=new Base();
	    }
	}

结果此时输出仍然为：  

	static initial block

	name=Tom age=48

	Base Constructor

如图3所示。这说明虽然初始化块执行的顺序由其在代码中的出现顺序决定，但是前提是相同属性的初始化块，即对于静态初始化块来说，它们的执行顺序是其出现的顺序；对于非静态初始化块来说，它们的执行顺序是其出现的顺序。但是对于不同属性的初始化块来说，静态初始化块总是比非静态初始化块先执行，而与它们在代码中出现的先后顺序无关。  

![java_init_block06](http://7xn1yt.com1.z0.glb.clouddn.com/java_init_block06.png)