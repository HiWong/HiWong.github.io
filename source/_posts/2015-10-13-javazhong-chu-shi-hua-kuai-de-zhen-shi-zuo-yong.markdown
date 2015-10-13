---
layout: post
title: "Java中初始化块的真实作用"
date: 2015-10-13 19:01:59 +0800
comments: true
categories: java
---
对于普通的类而言，可以放在初始化块中的初始化工作其实完全可以放到构造函数中进行，只不过有时会带来些许不便，如有多个构造器，就要在多个地方加上初始化函数完成初始化工作，而如果放到初始化块中的话则只要写一次即可。如下例所示<!--more-->：  

	import java.util.*;

	public class Car
	{
	   private void init()
	   {
	      System.out.println("Start engine before run()");
	   }
	    
	   public Car()
	   {
	      init();
	      System.out.println("Run normally");
	   }
	   
	   public Car(float velocity)
	   {
	      init();
	      System.out.println("Run with "+velocity+" miles/h");
	    }

	   public static void main(String[]args)
	   {
	      new Car();
	      new Car(210f);
	   }
	}

输出结果如下：  

![java_init01](http://7xn1yt.com1.z0.glb.clouddn.com/java_init01.png)

如果使用初始化块，则代码如下：  

	import java.util.*;

	public class Car
	{
	   private void init()
	   {
	      System.out.println("Start engine before run()");
	   }
	    
	   //使用初始化块
	   {
	     init();
	    }
	  

	   public Car()
	   {
	      //init();
	      System.out.println("Run normally");
	   }
	   
	   public Car(float velocity)
	   {
	      //init();
	      System.out.println("Run with "+velocity+" miles/h");
	    }

	   public static void main(String[]args)
	   {
	      new Car();
	      new Car(210f);
	   }
	}

显然，使用初始化块的代码比不使用初始化块的代码要更简洁。  
但是，如果只是这一点便利的话，还不足以使用初始化块，其实初始化块真正体现其独一无二的作用是在匿名内部类中，由于是匿名内部类，因而无法写构造方法，但是很多时候还是要完成相应的初始化工作，这时就需要用到初始化块了，特别是Android中大量地使用匿名内部类，初始化块的作用就十分突出，如下例所示： 

	import java.util.*;

	interface Vehicle
	{
	  public void run();
	  public void carry(); 
	 }

	class Person
	{
	  public void drive(Vehicle vehicle)
	  {
	    vehicle.run();
	   }
	}

	public class AnonymousSample
	{
	   public static void main(String[]args)
	   {
	      Person person=new Person();
	      person.drive(new Vehicle()
	      {
	        private void init()
	        {
	           System.out.println("Start engine befor run()");
	        }
	        
	        //匿名类中只能使用普通初始化块来完成初始化工作
	        {
	          init(); 
	         }

	        @Override
	        public void run()
	        {
	           System.out.println("Car can run with high velocity on freeway.");
	        }
	        @Override
	        public void carry()
	        {
	           System.out.println("Car can carry 3 persons at least.");
	         }
	      });
	   }
	}

输出结果如下：  

![java_init02](http://7xn1yt.com1.z0.glb.clouddn.com/java_init02.png)

**显然，在本例中，由于实现Vehicle的匿名内部类没有名字，自然也不能有显式的构造方法，从而无法在构造方法中完成初始化工作，但是如果不完成初始化就无法正常启动，所以此时就要借助初始化块。**  








