---
layout: post
title: " Java中的null对象也可以访问static成员变量和方法"
date: 2015-10-13 19:29:09 +0800
comments: true
categories: java
---
一般来说，一个类的对象要在实例化之后才可以访问类中的成员变量和方法。如果它还是null，通常意义上我们就认为它不能访问类中的成员。实际上确实不提倡这样，而且null对象确实不能访问实例成员(变量和方法)，否则会引发NULLPointerException错误。但是要注意的一点是：即使是null对象，也可以访问类成员。看下面一段代码的输出结果即知<!--more-->。  

	import java.util.*;

	public class Apple
	{
	  public static int weight=300;
	  
	  public static void print()
	  {
	    System.out.println("Weight is "+String.valueOf(weight));
	  }
	  public static void main(String[]args)
	  {
	    Apple apple=null;
	    System.out.println(apple.weight);
	    apple.print();
	  }
	}

输出结果如下图所示：
(注：csdn网站服务器好像出了点小问题，这会儿图片一直提交不上去，所以等它恢复了再追加吧，下面直接说运行结果。有兴趣的小伙伴不妨自己去验证一下。）  

刚刚发现图片可以上传了，下面是输出结果。  

![java_null01](http://7xn1yt.com1.z0.glb.clouddn.com/java_null01.png)

**运行结果不但没出现NULLPointerException错误，还输出了“300”及“Weight is 300"，这说明null对象apple可以调用类成员(即static成员）。  

而这显然是违背我们平常的使用习惯的，因而在C#中就干脆规定只有类才能调用类成员变量，对象只能调用实例成员(虽然很多人都说C#是抄袭Java的，但是不得不说在某些方面它做得更好）。

即使是针对Java，也有许多人在讨论是否将对象可以访问类成员这一功能取消掉，因为这个功能有时确实会给人带来困扰。  

远的不说，就举一个最近我经历的例子，前几天我在走读项目的代码时，发现有一个地方将对象初始化为null之后，马上就让其调用一个方法，我当时就觉得很奇怪，还以为是同事犯错了呢，后来经过沟通才知道。虽然这种做法是没错，但是显然会使代码不利于他人走读，从这方面来说并不是一个好的编码习惯。所以现在在团队里大家就约定类方法就用类来调用，而实例方法用对象来调用（当然这个不会犯错，因为类也没法调用实例方法）。**  

