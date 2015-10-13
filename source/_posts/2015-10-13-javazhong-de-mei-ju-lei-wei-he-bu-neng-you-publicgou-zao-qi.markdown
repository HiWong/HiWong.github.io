---
layout: post
title: "Java中的枚举类为何不能有public构造器"
date: 2015-10-13 18:58:57 +0800
comments: true
categories: java
---
从Java 5开始有了枚举类，需要注意的是enum定义的类默认继承的是java.lang.Enum类而不是Object类。同时注意枚举类不能派生子类（类的默认修饰符为final)，其原因基于它只有private构造器，那为什么要设计成这样呢？  

其实很容易想明白，所谓枚举类就是有包含有固定数量实例（并且实例的值也固定）的特殊类，如果其含有public构造器，那么在类的外部就可以通过这个构造器来新建实例，显然这时实例的数量和值就不固定了，这与定义枚举类的初衷相矛盾，为了避免这种形象，就对枚举类的构造器默认使用private修饰。如果为枚举类的构造器显式指定其它访问控制符，则会编译出错。  

另外，注意枚举类的所有实例必须在其首行显式列出，否则它不能产生实例。如下是一个使用枚举类的经典示例：  

	import java.util.*;

	enum Planet
	{
	  MERCURY,VENUS,EARTH,MARS,JUPITER,SATURN,URANUS,NEPTUNE
	}

	public class EnumSample
	{
	   public void flyTo(Planet planet)
	   {
	     String destination="";
	     switch(planet)
	     {
	       case MERCURY:
	          destination="水星";
	          break;
	       case VENUS:
	          destination="金星";
	          break;
	       case EARTH:
	          destination="地球";
	          break;
	       case MARS:
	          destination="火星";
	          break;
	       case JUPITER:
	          destination="木星";
	          break;
	       case SATURN:
	           destination="土星";
	           break;
	       case URANUS:
	           destination="天王星";
	           break;
	       case NEPTUNE:
	           destination="海王星";
	           break;
	      }
	      System.out.println("The destination is "+destination);
	    }
	  public static void main(String[]args)
	  {
	     EnumSample sample=new EnumSample();
	     sample.flyTo(Planet.NEPTUNE);
	   }
	}


