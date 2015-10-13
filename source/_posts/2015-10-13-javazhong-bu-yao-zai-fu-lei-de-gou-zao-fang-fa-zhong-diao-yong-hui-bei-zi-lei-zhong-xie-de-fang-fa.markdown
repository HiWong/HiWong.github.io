---
layout: post
title: "Java中不要在父类的构造方法中调用会被子类重写的方法"
date: 2015-10-13 20:02:50 +0800
comments: true
categories: java
---
在Java中，不要在父类的构造函数中调用会被子类重写的方法，否则运行时会遇到意想不到的错误。看一个例子就会明白<!--more-->：  

	import java.util.*;

	class Animal
	{
	   public Animal()
	   {
	     eat();
	   }
	   
	   protected void eat()
	   {
	     System.out.println("Eat something");
	   }
	}

	public class Bird extends Animal
	{
	  public Bird()
	  {
	   
	  }
	  @Override
	  protected void eat()
	  {
	    System.out.println("Just eat worm");
	  }
	  
	  public static void main(String[]args)
	  {
	    new Bird();
	  }
	} 

输出结果如下：  

![java_super01](http://7xn1yt.com1.z0.glb.clouddn.com/java_super01.png)

显然，在执行父类的构造方法时，调用的并非是父类中的eat()方法，而是子类中重写了的eat()方法。有人也许会怀疑是在父类中没有写上this.eat()，那么如果加上this,即代码如下所示：  

	import java.util.*;

	class Animal
	{
	   public Animal()
	   {
	     this.eat();
	   }
	   
	   protected void eat()
	   {
	     System.out.println("Eat something");
	   }
	}

	public class Bird extends Animal
	{
	  public Bird()
	  {
	    
	  }
	  @Override
	  protected void eat()
	  {
	    System.out.println("Just eat worm");
	  }
	  
	  public static void main(String[]args)
	  {
	    new Bird();
	  }
	} 
   
此时输出结果如下图所示：  

![java_super02](http://7xn1yt.com1.z0.glb.clouddn.com/java_super02.png)

显然，此时输出结果与上面的相同。那么原因是什么呢？  

这其实涉及到编译时类型和运行时类型的问题，在上面的例子中，虽然使用了this，即在调用父类的构造方法中，其编译时类型确实是Animal，但是运行时类型却是Bird，因而调用的是子类中重写的eat()方法。  

在上面的例子中，由于调用被子类重写的方法只是导致输出出错，严重的例子可能导致运行时错误，代码如下：  

	import java.util.*;

	class Animal
	{
	   public Animal()
	   {
	     this.eat();
	   }
	   
	   protected void eat()
	   {
	     System.out.println("Eat something");
	   }
	}

	public class Bird extends Animal
	{
	  private String wormType;
	  public Bird()
	  {
	    wormType="Leafworm";
	  }
	  @Override
	  protected void eat()
	  {
	    System.out.println("The length of wormType is "+wormType.length());
	  }
	  
	  public static void main(String[]args)
	  {
	    new Bird();
	  }
	} 

运行结果为：  

![java_super03](http://7xn1yt.com1.z0.glb.clouddn.com/java_super03.png)

由上图可知，在编译时未出现错误，但是在运行时却出现空指针错误，原因就在于wormType还没被初始化，从而调用String.length()方法是出错。

**综上可得到结论：在父类的构造方法中，如果要调用其他方法，绝对不能调用可能会被子类重写的方法，否则会出现意想不到的错误。这也就意味着，如果要父类的构造方法中要调用其他成员方法，那么要么调用private方法，要么调用final修饰的方法，因为它们都是不能被子类重写的方法。**

   

