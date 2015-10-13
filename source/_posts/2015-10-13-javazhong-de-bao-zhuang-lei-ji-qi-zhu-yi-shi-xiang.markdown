---
layout: post
title: "Java中的包装类及其注意事项"
date: 2015-10-13 19:53:40 +0800
comments: true
categories: java
---
我们知道，在Java中，除了8种基本数据类型外，其他的都是引用类型。使用引用类型是为了更好地贯彻面向对象的思想，那为什么还要保留8种基本数据类型呢？  

这其实更多地是照顾程序员的习惯。为了既照顾程序员的习惯，同时又能全面贯彻面向对象编程的思想，Java中引入了包装类机制。所谓的包装类就是为8种基本数据类型分别定义了相应的引用类型，其对应关系如下：  

![java_wrapper_class](http://7xn1yt.com1.z0.glb.clouddn.com/java_wrapper_class.png)

显然，除了int及char外，其余的包装类都是将对应的基本数据类型的首字母大写即可。
那为什么要引入包装类呢？前面已经说过，是为了全面贯彻面向对象的编程思想，具体地说就是非引用类型的数据在使用时会有许多制约，比如List list=new ArrayList();对于引用类型，可直接使用list.add(obj);进行添加，但是对于基本数据类型则无法添加，从而不能使用ArrayList中的许多方法（如排序、删除等），显然会造成许多不便，而使用包装类则可以很好地避免这种缺陷。  

同时，从JDK 1.5开始提供了自动装箱和自动拆箱的功能，因而目前可以有以下3种初始化包装类的方法：  

方法1：直接传入相应的基本数据类型变量或常量，如   

	int a1=3;Integer a2=new Integer(a1);
	Float f=new Float(3.14f);
	Boolean b=new Boolean(true);

方法2：通过传入字符串，如   

	Integer a=new Integer("3");
	Float f=new Float("3.14");
	Boolean b=new Boolean("true");

值得注意的是使用"True"也可以，如Boolean b=new Boolean("True");  

方法3：通过自动装箱功能，如Integer a=3;Float f=3.14f;Boolean b=true;值得注意的是可使用new Float("3.14")和new Float("3.14f")这样的语句来初始化Float类型变量，但是却不能使用Float f=3.14;来初始化Float类型变量，因为3.14是double类型，它只能被自动装箱为Double类型变量。  

我们知道，引用类型使用==进行比较时，只有当二者指向同一个对象时，才会返回true，否则即使值相等也返回false.包装类也属于引用类型，所以以下代码的执行结果为false,	  

	Float f1=new Float(3.14f);
	Float f2=new Float(3.14f);
	System.out.println(f1==f2);

但是，下面一段代码的输出结果却和前面讨论的不一样，这是为什么呢？

	import java.util.*;

	public class TestWrapperClass
	{
	   public static void main(String[]args)
	   {
	      Integer t1=3;
	      Integer t2=3;
	      System.out.println(t1==t2);
	      
	      Integer t3=128;
	      Integer t4=128;
	      System.out.println(t3==t4);
	      
	      Boolean b1=true;
	      Boolean b2=true;
	      System.out.println(b1==b2);
	    }
	}

其输出结果如下图所示：  

![java_wrapper_class02](http://7xn1yt.com1.z0.glb.clouddn.com/java_wrapper_class02.png)

如果按照前面的讨论，应该都输出false才对，但这里t1与t2,b1与b2的比较结果却为true.这不科学啊！  

原来，Java为了获得更高的执行效率，在某些类的设计中引入了缓存机制!  

此处的Integer及Boolean类的设计即是如此。java.lang.Integer类的部分源代码如下所示：  

	static final Integer[]cache=new Integer[-(-128)+127+1];
	static{
	for(int i=0;i<cache.length;i++)
	   cache[i]=new Integer[i-128);
	}
	
显然，系统把-128~127之间的整数装箱成Integer实例，并通过cache数组进行缓存，所以只要是-128~127之间的Integer类型变量，其指向的对象都是cache数组成员，从而只要有两个值相同且在-128~127之间的Integer变量，它们指向的对象就是同一个，故采用==进行比较时也返回true.Boolean的情形与之类似。  

实际上，不只是在Java中，在Android中的一些类也采用了缓存机制，如Android中的ListView就是一个典型的例子，在继承的方法getView中，convertView其实就是采用了缓存机制，从而大大节省了系统资源开支，加快了图形渲染的速度。此处暂且不表，在后面还会再提到。  
