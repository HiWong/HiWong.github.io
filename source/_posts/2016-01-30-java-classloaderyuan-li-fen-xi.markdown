---
layout: post
title: "Java ClassLoader原理分析"
date: 2016-01-30 14:56:04 +0800
comments: true
categories: Android内核分析
---

1.认识ClassLoader  

Java中的所有类，必须被装载到jvm中才能运行，这个装载工作是由jvm中的类装载器完成的，类装载器所做的工作实质是把类文件从硬盘读取到内存中，JVM在加载类的时候，都是通过ClassLoader的loadClass()方法来加载class的。需要注意的是，程序在启动的时候，并不会一次性加载所有的class文件，而是根据需要，通过ClassLoader来动态加载<!--more-->相应的class文件到内存中。  

2.Bootstrap ClassLoader, Extension ClassLoader, App ClassLoader  

先看下面一段代码：  

	public class ClassLoaderSample{
		public static void main(String[]args){
			ClassLoader loader=ClassLoaderSample.class.getClassLoader();
			while(loader!=null){
				System.out.println(loader);
				loader=loader.getParent();
			}
			System.out.println(loader);
		}
	}

代码执行结果如下：  

![ClassLoaderParent](http://7xn1yt.com1.z0.glb.clouddn.com/ClassLoaderParent.png)

通过输出可看到通过ClassLoaderSample类直接获得的类加载器是App ClassLoader，而App ClasLoader的父类是Extension ClassLoader，但是Extension ClassLoader的父类却是null,实际上最顶层的是Bootstrap ClassLoader，只不过它是由C++写的，负责加载JDK中的核心类库。而这样设计的原因是涉及到一个类似操作系统中"鸡生蛋，蛋生鸡"的问题，因为所有的类都是通过ClassLoader装载，可是ClassLoader本身也是一个类，那第一个ClassLoader由谁来负责装载？实际上这部分工作就是由Bootstrap ClassLoader来完成的，类似于操作系统启动时的boot loader.  

1)Bootstrap ClassLoader  

Bootstrap ClassLoader称为启动加载器，是Java类中加载层次最顶层的类加载器，负责加载JDK中的核心类库，如rt.jar,resources.jar,charsets.jar等；另外，在命令行中用-Xbootclasspath选项指定的jar包也是通过Bootstrap ClassLoader加载。  

看下面一段代码：  

	public static void main(String[]args){
		URL[]urls=sun.misc.Launcher.getBootstrapClassPath().getURLs();
		for(int i=0;i<urls.length;++i){
			System.out.println(urls[i].toExternalForm());
		}
	}

输出结果如下：  

![Bootstrap ClassLoader](http://7xn1yt.com1.z0.glb.clouddn.com/BootstrapClassLoader.png)

2)Extension ClassLoader  

扩展类加载器，负责加载Java的扩展类库，默认加载JAVA_HOME/jre/lib/ext/目录下的所有jar;另外，用-Djava.ext.dirs指定目录下的jar包也是通过Extension ClassLoader加载。  

3)App ClassLoader  

系统类加载器，负责加载应用程序classpath目录下的所有jar和class文件；另外，通过-Djava.class.path所指定的目录下的类和jar包也是通过它加载；  

3.ClassLoader加载原理  

ClassLoader使用双亲委托来加载类，所谓双亲委托，就是当一个ClassLoader实例需要加载某个类时，它会先将这个任务委托给它的父类加载器（实际上最先在缓存中查找，不过并不影响原理的理解，我们这里先不讨论)，而父类加载器又会将任务委托给它的父类加载器，依次循环下去。所以类的加载过程是自顶向下的，即首先由最顶层的Bootstrap ClassLoader试图加载，如果没有加载到，再把任务交给Extension ClassLoader,如果还没有加载到，则转交给App ClassLoader，如果App ClassLoader也没有加载到的话，则返回给委托的发起者(比如用户自定义的ClassLoader),由它到指定的文件系统或网络中加载该类。如果最终没有一个加载成功，则抛出ClassNotFoundException异常。  
前面已经说过，在类的加载过程中实际上还使用了缓存，将缓存考虑进去的话，则完整的类加载过程如下：
1)当前ClassLoader首先从自己已经加载的类中查询是否已经加载，如果已经加载则直接返回原来已经加载的类。实际上，每个类加载器都有自己的加载缓存，当一个类通过其加载时，就会放入其缓存中，下次就可以直接从缓存中获取。  
2)当前ClassLoader的缓存中没有找到被加载类的时候，委托父类加载器去加载，而父类加载器也采用同样的策略，首先查看自己的缓存，如果缓存中没有则委托其父类去加载，一直到Bootstrap ClassLoader;  
3)当所有的父类加载器都没有加载的时候，再交由委托的发起者进行加载，并将其放入它自己的缓存中，以便下次有加载请求的时候直接返回。如果最后所有的ClassLoader都没有加载成功，则抛出ClassNotFoundException.  

这个加载过程的示意图如下：  

![ClassLoaderFlow](http://7xn1yt.com1.z0.glb.clouddn.com/ClassLoaderFlow.png)

4.采用双亲委托的原因  

采用双亲委托可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子类ClassLoader现加载一次。试想一下，如果不使用这种委托模型，那用户就可以随时使用自定义的String来动态代替jdk中的String，这样就会有很大的安全隐患。而采用双亲委托的方式，即使用户自定义了ClassLoader，也可以避免这种情况发生，除非篡改jdk中ClassLoader搜索类的默认算法之后再打包.所以jdk和jre一定要从官方获取，不然就可能发生2015年出现的xcode来源导致的用户信息泄露问题。  

7.利用ClassLoader进行动态远程加载  

从上面的分析可知，利用ClassLoader可以动态加载类，不仅是从本地，还可以从远程服务器上加载。  
首先是服务端的代码：  
    
    package wang.imallen.blog;
	public class Candy {

		public void display()
		{
			System.out.println("I'm marshmallow instead of Lollipop");
		}
	}

然后将编译生成的Candy.class部署到wang/imallen/blog的目录下。  

下面是客户端的代码，为了从服务端加载我们需要的类，我们自定义了一个ClassLoader，代码如下：  

	import java.io.ByteArrayOutputStream;
	import java.io.IOException;
	import java.io.InputStream;
	import java.net.URL;

	public class NetworkClassLoader extends ClassLoader
	{

		private static final String TAG=NetworkClassLoader.class.getSimpleName();
		
		private String url;
		
		public NetworkClassLoader(String url)
		{
			this.url=url;
		}
		
		@Override
		protected Class<?>findClass(String name)throws ClassNotFoundException
		{
			
			Class clazz=null;
			byte[]classData=downloadClassFile(name);
			if(classData==null)
			{
				throw new ClassNotFoundException();
			}
			clazz=defineClass(name,classData,0,classData.length);
			
			return clazz;	
		}
		
		private byte[]downloadClassFile(String name)
		{
			InputStream is=null;
			try
			{
				String path=className2Path(name);
				URL url=new URL(path);
				byte[]buff=new byte[1024];
				int len=-1;
				is=url.openStream();
				ByteArrayOutputStream baos=new ByteArrayOutputStream();
				while((len=is.read(buff))!=-1)
				{
					baos.write(buff,0,len);
				}	
				System.out.println("now we have downloaded the class file");
				return baos.toByteArray();
			}
			catch(Exception ex)
			{
				ex.printStackTrace();
			}
			finally
			{
				if(is!=null)
				{
					try
					{
						is.close();
					}
					catch(IOException ex)
					{
						ex.printStackTrace();
					}
				}
			}
			
			return null;
		}
		
		private String className2Path(String name)
		{
			return url+"/"+name.replace(".", "/")+".class";
		}
	}

下面是测试代码，为简便起见，也写在NetworkClassLoader中：  

	public static void main(String[]args)
	{
		String rootUrl="http://192.168.2.102:8080/Test";
		String className="wang.imallen.blog.Candy";
		NetworkClassLoader loader=new NetworkClassLoader(rootUrl);
		
		try
		{
			Class<?>clazz=loader.loadClass(className);
			Object obj=clazz.newInstance();
			clazz.getMethod("display").invoke(obj);	
		}
		catch(Exception ex)
		{
			ex.printStackTrace();
		}
		
	}

下面是测试结果：  

![DynamicClassLoader](http://7xn1yt.com1.z0.glb.clouddn.com/DynamicLoader.png)

上面这种方法其实有非常重要的应用，比如对已经发布的应用进行热补丁修复，或者是动态地添加新功能，而不需要重新发布，这也是我们需要自定义ClassLoader的一个重要原因。  

6.关于ClassLoader的一个细节问题  

JVM在判定两个class是否相同时，不仅要判断两个类名是否相同，还要判断是否由同一个类加载器实例加载的。只有在两者都满足的情况下，JVM才认为这两个class是相同的。否则，即使是同一个class文件，如果被两个不同的ClassLoader实例所加载，JVM也会认为它们是两个不同的类。要验证也很简单，我们只需要将在Candy中增加如下方法之后重新部署：  

	public void test(Object obj)
	{
			if(obj instanceof UCandy)
			{
				System.out.println("Same class");
			}
			else
			{
				System.out.println("Different classes");
			}
	}

而测试代码如下所示：  

	public static void main(String[]args)
	{	
		String rootUrl="http://192.168.2.102:8080/Test";
		String className="wang.imallen.blog.Candy";
		NetworkClassLoader loader1=new NetworkClassLoader(rootUrl);
		NetworkClassLoader loader2=new NetworkClassLoader(rootUrl);
		
		try
		{
			Class<?>clazz1=loader1.loadClass(className);
			Class<?>clazz2=loader2.loadClass(className);
			Object obj1=clazz1.newInstance();
			Object obj2=clazz2.newInstance();
			clazz1.getMethod("test",Object.class).invoke(obj1,obj2);	
		}catch(Exception ex)
		{
			ex.printStackTrace();
		}
			
	}	

测试结果如下：  

![DifferentClasses](http://7xn1yt.com1.z0.glb.clouddn.com/DifferentClasses.png)

显然由于使用的ClassLoader不同，导致加载出来的类也不同。

