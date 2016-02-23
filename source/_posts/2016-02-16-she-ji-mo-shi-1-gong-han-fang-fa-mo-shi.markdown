---
layout: post
title: "设计模式(1):工厂方法模式"
date: 2016-02-16 01:12:31 +0800
comments: true
categories: design_pattern
---


1.工厂方法模式  
工厂方法模式分为三种:普通工厂模式,多个工厂方法模式,静态工厂方法模式.
1.1普通工厂模式  
所谓普通工厂模式,就是建立一个工厂类,对实现了同一接口的一些类进行实例的创建。我们以生产糖果为例，一个工厂生产各种口味糖果，用户需要哪种糖果，只需要告知对应糖果的名称即可。UML图如下:<!--more-->

首先，我们创建二者的共同接口:  

	public interface Candy{
		public void flavor();
	}

然后，创建3种糖果类:  

	public class Cupcake implements Candy{
		@Override
		public void flavor()
		{
			System.out.println("I'm cupcake.")
		}
	}

	public class Donut implements Candy{
		@Override
		public void flavor(){
			System.out.println("I'm donut.")
		}
	}

	public class Eclair implements Candy{
		@Override
		public void flavor(){
			System.out.println("I'm eclair.");
		}
	}

最后,建立工厂类.

	public class CandyFactory{

		public static final String CUP_CAKE="cupcake";
		public static final String DONUT="donut";
		public static final String ECLAIR="eclair";

		public Candy produce(String type){
			if(CUP_CAKE.equals(type)){
				return new Cupcake();
			}else if(DONUT.equals(type)){
				return new Donut();
			}else if(ECLAIR.equals(type)){
				return new Eclair();
			}

		}
	}

其实CandyFactory应该设计为单例模式，到后面讲解到单例模式时再展开来说，这里就先这样。  

下面，我们进行测试:
	
	public class FactoryTest{
		public static void main(String[]args){
			CandyFactory factory=new CandyFactory();
			Candy cupcake=factory.produce(CandyFactory.CUP_CAKE);
			Candy donut=factory.produce(CandyFactory.DONUT);
			Candy eclair=factory.produce(CandyFactory.ECLAIR);

			cupcake.flavor();
			donut.flavor();
			eclair.flavor();
		}
	}

运行后输出不同口味的语句，显然，利用工厂模式进行了较好的封装。  

1.2多个工厂方法模式  
在普通工厂方法模式中，如果传递的字符串出错，就不能正确创建对象，而多个工厂方法模式则是在工厂类中提供多种方法来创建对象。UML图如下:

采用多个工厂方法模式时，CandyFactory的代码如下：  

	public class CandyFactory{
		public Candy produceCupcake(){
			return new Cupcake();
		}

		public Candy produceDonut(){
			return new Donut();
		}

		public Candy produceEclair(){
			return new Eclair();
		}
	}

1.3静态工厂方法模式  
所谓静态工厂方法模式，就是将上面的多个工厂方法模式里的方法置为静态的，不需要创建实例，直接调用即可，这样就不必创建工厂类对象。  
代码如下所示：  

	public class CandyFactory{
		public static Candy produceCupcake(){
			return new Cupcake();
		}
		public static Candy produceDonut(){
			return new Donut();
		}
		public static Candy produceEclair(){
			return fnew Eclair();
		}
	}

2.抽象工厂模式(Abstract Factory)
工厂方法模式有一个问题就是，类的创建依赖工厂类，也就是说，如果想要拓展程序，必须对工厂类进行修改，这违背了闭包原则，所以，从设计角度考虑，有一定的问题，如何解决？这需要用到抽象工厂模式，即创建多个工厂类，这样一旦需要增加新的功能，直接增加校报的工厂类就可以了，不需要修改之前的代码。UML图如下所示：  

首先定义Provider:  

	public interface Provider{
		public Candy produce();
	}

下面是两个糖果的实现类：  

	public class CupcakeFactory implements Provider{
		@Override
		public Candy produce(){
			return new Cupcake();
		}	
	}

	public class DonutFactory implements Provider{
		@Override
		public Candy produce(){
			return new Donut();
		}
	}

	public class EclairFactory implements Provider{
		@Override
		public Candy produce(){
			return new Eclair();
		}
	}

下面是测试代码:  

	public class Test{
		public static void main(String[]args){
			Provider cupcakeProvider=new CupcakeFactory();
			Provider donutProvider=new DonutFactory();
			Provider eclairProvider=new EclairFactory();

			Candy cupcake=cupcakeProvider.produce();
			Candy donut=donutProvider.produce();
			Candy eclair=eclairProvider.produce();

			cupcake.flavor();
			donut.flavor();
			eclair.flavor();
		}
	}