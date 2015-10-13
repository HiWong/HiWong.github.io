---
layout: post
title: "深刻理解Java中形参与实参，引用与对象的关系"
date: 2015-10-13 17:27:26 +0800
comments: true
categories: java
---
我们都知道，在Java中，除了基本数据类型之外，其他的都是引用类型，当它们作为函数参数时，传递的也是引用，通过引用可以改变对象的值，很多人便因此而忽略形参与实参，引用与对象的关系问题。废话不多说，先看下面一个例子<!--more-->：  

	import java.util.*;

	public class Student
	{
	   private String name;
	   private int age;
	   public Student(String name,int age)
	   {
	      this.name=name;
	      this.age=age;
	   }
	   public void printInfo()
	   {
	      System.out.println("Name:"+name+" age:"+age);
	    }
	   
	   public static void change(Student student)
	   {
	      Student stu=new Student("Lucy",26);
	      student=stu;
	      student.printInfo();
	    }
	   public static void main(String[]args)
	   {
	      Student s=new Student("Lily",25);
	      s.printInfo();
	      change(s);
	      s.printInfo();
	    }
	}

运行结果如下：  

![java_param01](http://7xn1yt.com1.z0.glb.clouddn.com/java_param01.png)

显然，形参改变了，但是实参没有被改变，这是为什么呢？  

要分析这个问题，首先要分清Java中的引用与对象的关系，比如此处s是引用，它存储在栈中，而它指向的对象存储在堆中，所以当使用change函数时，实际上只进行了以下操作：首先是将实参的值传给形参，即形参student也指向s所指向的对象；之后让student指向change函数中新建的对象，也就是说，在这个函数中，实参只是在一开始起了一下作用，后面便再也没它的事儿了。我们增加一个输出操作即可印证上面的结论。  
新代码如下：  

	import java.util.*;

	public class Student
	{
	   private String name;
	   private int age;
	   public Student(String name,int age)
	   {
	      this.name=name;
	      this.age=age;
	   }
	   public void printInfo()
	   {
	      System.out.println("Name:"+name+" age:"+age);
	    }
	   
	   public static void change(Student student)
	   {
	      //新增的输出操作
	      if(student!=null)
	      {
	         student.printInfo();
	      }
	      Student stu=new Student("Lucy",26);
	      student=stu;
	      student.printInfo();
	    }
	   public static void main(String[]args)
	   {
	      Student s=new Student("Lily",25);
	      s.printInfo();
	      change(s);
	      s.printInfo();
	    }
	}

输出结果如下：  

![java_param02](http://7xn1yt.com1.z0.glb.clouddn.com/java_param02.png)

由输出结果可知确实是实参只起了一次结合的作用，而在整个函数中实参s及其指向的对象都未被改变，所以显然change函数达不到相应的预期。实际上，要实现改变对象的目的，应该像下面这样写：  

	import java.util.*;

	public class Student
	{
	   private String name;
	   private int age;
	   public Student(String name,int age)
	   {
	      this.name=name;
	      this.age=age;
	   }
	   public void printInfo()
	   {
	      System.out.println("Name:"+name+" age:"+age);
	    }
	    public void setName(String name)
	    {
	       this.name=name;
	    }
	    public void setAge(int age)
	    {
	       this.age=age;
	    }
	    public static void change(Student student)
	    {
	      student.setName("Lucy");
	      student.setAge(26);
	    }
	    public static void main(String[]args)
	    {
	      Student s=new Student("Lily",25);
	      s.printInfo();
	      change(s);
	      s.printInfo();
	     }
	}

输出结果如下：  

![java_param03](http://7xn1yt.com1.z0.glb.clouddn.com/java_param03.png)

显然，这个change函数达到了我们的预期，原因就在于它是真正地改变了对象的属性。  

**通过上面这个例子可知，虽然Java中传递的是引用，可以轻易地实现对对象的改变，但是仍然要注意形参与实参、引用与对象的关系，千万不要简单地以为传引用就一定可以实现对象的改变，否则可能犯下低级错误。**


