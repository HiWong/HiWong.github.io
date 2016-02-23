---
layout: post
title: "设计模式(2):单例模式"
date: 2016-02-16 01:18:28 +0800
comments: true
categories: design_pattern
---

1.什么是单例模式(Singleton)    
单例模式是一种常用的设计模式。在Java应用中，单例对象能保证在一个JVM中，该对象只有一个实例存在。这样的模式有几个好处:  
1)某些类的创建比较频繁，对于一些大型的对象，这是一笔很大的系统开销;  
2)省去了new操作符,降低了系统内存的使用频率,减轻GC压力;  
3)有些类如交易所的核心交易引擎，控制着交易流程，如果该类可以创建多个的话，系统就完全乱了。另外比如ImageLoader,加载图片的引擎也只需要一个。像这些情形都需要使用单例模式。

2.单例模式的使用  
首先是一个简单的示例<!--more-->:  

	public class ImageEngine{
		private static ImageEngine instance=null;

		private ImageEngine(){

		}

		public static ImageEngine getInstance(){
			if(instance==null){
				instance=new ImageEngine();
			}
			return instance;
		}

		public Object readResolve(){
			return instance;
		}
	}

这个类可用于单线程的情形，但是如果是要用在多线程情形中，则显然不适合.如何解决？可能大多数人想到的解决方法就是加synchronized关键字，如下所示:  

	public static synchronized ImageEngine getInstance(){
		if(instance==null){
			instance=new Singleton();
		}
		return instance;
	}
虽然做到了线程安全，但是并不高效。因为在任何时候只能有一个线程调用getInstance()方法。但是实际上同步操作只需要在第一次调用时才需要。为了解决这个问题，就需要使用双重检验锁。  

3.双重检验锁  
双重检验锁模式(Double Checked Locking Pattern)是一种使用同步加锁的方法。程序员称其为双重检验锁，因为会有两次检查instance==null，一次是在同步块外，一次是在同步块内。为什么在同步块内还要检验一次？因为可能会有多个线程一起进入同步块外的if，如果在同步块内不进行二次检验的话就会生成多个实例了。下面是使用双重检验锁的代码：  

	public static ImageEngine getInstance(){
		if(instnce==null){
			synchronized(ImageEngine.class){
				if(instance==null){
					instance=new ImageEngine();
				}
			}
		}
		return instance;
	}

上面这段代码看起来很完美,但是实际上还是有问题。主要在于instance=new ImageEngine()这句，它并非一个原子操作，事实上在JVM中这句话大概做了下面3件事情：
(1)给instance分配内存;
(2)调用ImageEngine的构造函数来初始化成员变量;
(3)将instance对象指向分配的内存空间(执行完这步instance就不为null了)

但是在JVM的即时编译器中存在指令重排序的优化。也就是说上面的第二步和第三步的顺序是不能保证的，最终的执行程序可能是1-2-3也可能是1-3-2，如果是后者,则在3执行完毕、2未执行之前，被另一个线程抢占了，则instance已经不是null了(但却没有初始化)，所以线程二会直接返回这个instance然后使用，但是由于它还没有初始化，所以会出错。  

为了解决这个问题，我们只要将instance变量声明成volatile就可以了。  

	public class ImageEngine{
		private volatile static ImageEngine instance;

		private ImageEngine(){}

		public static ImageEngine getInstance(){
			if(instance==null){
				synchronized(ImageEngine.class){
					if(instance==null){
						instance=new ImageEngine();
					}
				}
			}
		}
	}

有些人认为使用volatile的原因是可见性，也就是保证线程在本地不会存有instance的副本，每次都是去主内存中读取。但其实是不对的。使用volatile的主要原因是禁止指令重新排序优化。也就是说，在volatile变量的赋值操作后面会有一个内存屏障(生成的汇编代码上),读操作不会被重排序到内存屏障之前。比如上面的例子，取操作比较在执行完1-2-3之后或者1-3-2之后，不存在执行到1-3然后就取到值的情况。从"先行发生原则"的角度理解的话，就是对于一个volatile变量的写操作都先行发生于后面对这个变量的读操作.  

但是要注意的是在Java5以前的版本中即使使用volatile的双检锁也还是有问题的，原因是Java5以前的JMM(Java内存模型)是存在缺陷的，即使将变量声明成volatile也不能完全避免重排序。这个volatile屏蔽重排序的问题在Java5中才得以修复，所以在这之后才可以放心使用volatile.

那肿么办，还能愉快地使用单例模式吗？  
其实还是有一些方法的，归纳起来有以下方法:
1)饿汉式static final field  
这种方法非常简单粗暴，因为单例的实例被声明成static final变量了，在第一次加载类到内存中时就会初始化，所以创建实例本身是线程安全的。代码示例如下：  

	public class ImageEngine{
		private static final ImageEngine instance=new ImageEngine();

		private ImageEngine(){}

		public static ImageEngine getInstance(){
			return instance;
		}
	}

当然，这种方法的缺点就是它并不是一种懒加载模式，单例会在加载类后一开始就被初始化，即使客户端没有调用getInstance()方法。这样就可能导致初始加载时耗费很长的时间，这在某些情形下是不可接受的。另外，饿汉式的创建方式在一些场景中是无法使用的，譬如ImageEngine实例的创建是依赖参数或者配置文件的，在getInstance()之前必须调用某个方法设置参数给它，那样这种方法就行不通了。  

2)静态内部类  
这种方法比较好，它也是<<Effective Java>>上所推荐的。  

	public class ImageEngine{
		private static ImageEngineHolder{
			private static final ImageEngine INSTANCE=new ImageEngine();
		}

		private ImageEngine(){}

		public static final ImageEngine getInstance(){
			return ImageEngineHolder.INSTANCE;
		}
	}

这种写法仍然使用JVM本身机制保证了线程安全问题。由于ImageEngineHolder是私有的，除了getInstance()之外没有办法访问它，因此它是饿汉式的；同时读取实例的时候不会进行同步，没有性能缺陷，而且也不依赖JDK版本。  

3)枚举  
用枚举写单例实在是太简单了，这也是它最大的优点。代码如下所示:  

	public enum ImageEngine{
		INSTANCE;
	}

我样可以通过ImageEngine.INSTANCE来访问实例，这比调用getInstance()方法简单多了。所以不需要担心double checked locking，而且还能防止反序列化导致重新创建新的对象。但是实际中很少看到有人这样写，因为enum其实跟JDK有关，在早期的JDK版本中并没有枚举类的实现。  