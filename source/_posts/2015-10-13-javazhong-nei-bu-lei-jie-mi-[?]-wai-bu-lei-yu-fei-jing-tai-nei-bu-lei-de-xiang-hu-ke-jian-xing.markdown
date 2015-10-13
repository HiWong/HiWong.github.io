---
layout: post
title: "Java中内部类揭秘(一):外部类与非静态内部类的'相互可见性'"
date: 2015-10-13 19:09:25 +0800
comments: true
categories: java
---
我们都知道，非静态内部类可以访问外部类的私有成员（包括私有变量和方法），这也正是使用非静态内部类与普通类的一个重要区别：非静态内部类是依赖于外部类对象而存在的，这种依赖就包括它要能自由地访问外部类对象的所有成员（因为private成员都可以访问了，其他权限的成员更不在话下。不过一般来说一个内部类只会访问外部类的部分成员而不是全部）。比如<!--more-->心脏作为单独的一个类存在可能没有太大的意义，它必须依附于具体的Person对象存在才有意义，而且心脏它要能够自由地访问Person对象的一些成员，如血液、营养等。  

显然，外部类对于非静态内部类而言是完全透明的。但是实际上，外部类与非静态内部类的另一个特征虽然不常用，却也值得注意，那就是非静态内部类其实跟外部类的其他成员类似，只是它的一个成员而已，因而即使非静态内部类的修饰符为private、即使非静态内部类的构造器修饰符为private，外部类也可以新建非静态内部类的对象。如下例所示：  

	import java.util.*;

	class Car
	{
	  private float gasAmount;
	  private String gasType;
	  public Car(float gasAmount,String gasType)
	  {
	    this.gasAmount=gasAmount;
	    this.gasType=gasType;
	    new Engine();
	  }

	  private void print(String msg)
	  {
	     System.out.println(msg);
	   }
	  private class Engine
	  {
	    private int rotateSpeed;
	    private Engine()
	    {
	      if(gasType=="93#"&&gasAmount>0)
	      {
	          rotateSpeed=1500;
	          print("Gas amount is "+String.valueOf(gasAmount)+" gallon now.Engine starts successfully");
	      }
	      else if(gasType=="93#"&&gasAmount<=0)
	      {
	          rotateSpeed=0;
	          print("Engine starts failed! Please add fuel first!");
	       }
	      else
	      {
	          rotateSpeed=0;
	          print("Gas type is not correct!");
	       }
	      
	    }
	  }
	}


	public class OuterSample
	{
	  public static void main(String[]args)
	  {
	     new Car(2.0f,"93#");
	     new Car(0.0f,"93#");
	  }
	}

输出结果如下图所示：  

![java_class01](http://7xn1yt.com1.z0.glb.clouddn.com/java_class01.png)

显然，由输出结果可看出：第一，虽然非静态内部类的修饰符和构造器均为private，但是外部类仍然可以创建内部类对象；第二，非静态内部类可以使用外部类的private成员（如此处的private成员变量gasType及gasAmount);  

另一个常常被人忽略的地方是：在外部类的方法中，也可以通过创建非静态内部类的对象来访问内部类包括private成员在内的所有成员，不过注意必须是外部类的实例成员才行，而不能在外部类的静态成员（包括静态方法和静态初始化块）中使用非静态内部类，原因很简单：非静态内部类可看作是外部类的一个实例成员，而静态成员不能访问实例成员。如下例所示：  

	import java.util.*;

	class Car
	{
	  private float gasAmount;
	  private String gasType;
	  private Engine engine;
	  public Car(float gasAmount,String gasType)
	  {
	    this.gasAmount=gasAmount;
	    this.gasType=gasType;
	    engine=new Engine();
	  }

	  public void printRotateSpeed()
	  {
	    //其实写成print("Rotate speed is "+String.valueOf(new Engine().rotateSpeed));也行，但是不太符合实际，因为一车对应一引擎
	     print("Rotate speed is "+String.valueOf(engine.rotateSpeed));
	   }
	  private void print(String msg)
	  {
	     System.out.println(msg);
	   }
	  private class Engine
	  {
	    private int rotateSpeed;
	    private Engine()
	    {
	      if(gasType=="93#"&&gasAmount>0)
	      {
	          rotateSpeed=1500;
	          print("Gas amount is "+String.valueOf(gasAmount)+" gallon now.Engine starts successfully");
	      }
	      else if(gasType=="93#"&&gasAmount<=0)
	      {
	          rotateSpeed=0;
	          print("Engine starts failed! Please add fuel first!");
	       }
	      else
	      {
	          rotateSpeed=0;
	          print("Gas type is not correct!");
	       }
	      
	    }
	  }
	}


	public class OuterSample
	{
	  public static void main(String[]args)
	  {
	     Car car01=new Car(2.0f,"93#");
	     car01.printRotateSpeed();
	     Car car02=new Car(0.0f,"93#");
	     car02.printRotateSpeed();
	  }
	}

输出结果如下图：  

![java_class02](http://7xn1yt.com1.z0.glb.clouddn.com/java_class02.png)

从输出结果可以看出，在外部类的方法printRotateSpeed()中，通过非静态内部类的对象来a访问了其private成员rotateSpeed,这其实跟实际中的情况很像，即发动机从汽车处获得燃料信息，汽车再从发动机处获得转速并显示在仪表盘上。  

综上，非静态内部类可自由访问外部类包括privated成员在内的所有成员，外部类也可通过创建内部类的对象来访问其包括private成员在内的所有成员，所以它们虽然在类层面不是相互可见的，但是从广义上来说具有相互可见性，这也是我在题目上打上双引号的原因。  


