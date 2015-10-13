---
layout: post
title: "Java中类的初始化块问题(二)"
date: 2015-10-13 20:16:53 +0800
comments: true
categories: java
---
在Java中类的初始化块问题(一)一文中，我已经阐述过Java中的静态初始化块、非静态初始化块及构造函数的执行顺序问题，但是那篇文章中没有涉及类的继承中初始化块的执行顺序，本文将重点探讨。  

为了能减少读者看代码的时间，本文将采用Java中类的初始化块问题(一)中Base作为基类(只是略有改动)，从而代码如下<!--more-->：  

	import java.util.*;

	class Base
	{
	  private String name="Tom";
	  private int age=48;
	  
	  {
	    System.out.println("name="+name+" age="+String.valueOf(age));
	   }

	  static
	  {
	    System.out.println("static initial block in Base");
	   }


	   public Base()
	   {
	      System.out.println("Base Constructor");
	   }
	}

	public class Sub extends Base
	{
	   private String hobby="Soccer";
	   
	   {
	      System.out.println("hobby is "+hobby);
	   }

	    static
	    {
	       System.out.println("static initial block in Sub");
	    }
	   
	    public Sub()
	    {
	      System.out.println("Sub Constructor");
	     }
	    
	    public static void main(String[]args)
	    {
	       Sub s;
	       //s=new Sub();
	     }
	}

此时输出结果如下图所示：  

![java_init_block01](http://7xn1yt.com1.z0.glb.clouddn.com/java_init_block01.png)

显然，此时由于只有基类和子类被加载到内存，但是并未创建对象，因而此时只有基类和子类的静态初始化块被执行。  

如果将上面代码中的s=new Sub();的注释符去掉，则此时输出结果如下图所示：  

![java_init_block02](http://7xn1yt.com1.z0.glb.clouddn.com/java_init_block02.png)

显然，此时执行顺序为：基类的静态初始化块--子类的静态初始化块--基类的非静态初始化块--基类的构造函数--子类的非静态初始化块--子类的构造函数。  

但是为什么是这样的执行顺序呢？因为首先类被加载到内存，因而先输出基类和子类的静态初始化块，然后由于要生成子类对象，因而非静态初始化块也被执行。从而执行基类的非静态初始化块之后再执行其构造函数。同理，在子类的构造函数被执行之前要先执行其非静态初始化块。  

其实这里涉及到类的初始化问题。一个类要被使用，需要经过装载、连接和初始化过程(这里所谓的“初始化”与上面不同，其实是狭义的初始化，特指非静态成员的初始化）。下面是这三个过程的含义。  

装载：类装载器把编译形成的class文件载入内存，创建类相关的Class对象，它里面封装了相应类的类型信息；  

连接：连接又可分为三个阶段，即验证、准备和解析。验证就是要确保数据格式的正确性，并适合JVM使用；准备即JVM为静态变量分配内存空间，并设置默认值；解析过程就是在类型的常量池中寻找类、接口、字段和方法的符号引用，把其替换成直接引用。  

从而，对于一个类来说，如果不生成该类的对象，那么只会在连接中的准备阶段执行其静态成员(包括静态变量和静态初始化块）。注意这个过程与是否生成对象无关，所以即使是下面这段代码，也会有相应的输出。  

	import java.util.*;

	class Base
	{
	  private String name="Tom";
	  private int age=48;
	  
	  {
	    System.out.println("name="+name+" age="+String.valueOf(age));
	   }

	  static
	  {
	    System.out.println("static initial block in Base");
	   }


	   public Base()
	   {
	      System.out.println("Base Constructor");
	   }
	}

	public class Sub extends Base
	{
	   private String hobby="Soccer";
	   
	   {
	      System.out.println("hobby is "+hobby);
	   }

	    static
	    {
	       System.out.println("static initial block in Sub");
	    }
	   
	    public Sub()
	    {
	      System.out.println("Sub Constructor");
	     }
	    
	    public static void main(String[]args)
	    {
	       
	     }
	}

此时输出结果如下图所示：  

![java_init_block03](http://7xn1yt.com1.z0.glb.clouddn.com/java_init_block03.png)

显然，虽然此时main函数为空，但是由于类在准备阶段会执行静态成员，因而会有上面的输出。  

如果需要生成实例化的对象，那么除了上面两个阶段，还要经过非静态成员的初始化过程（其顺序为非静态成员在类中出现的顺序)。至此，我们就可以完美解释图2中的输出了。  
