---
layout: post
title: "深刻理解Java中final的作用(一):从final的作用剖析String被设计成不可变类的深层原因"
date: 2015-10-13 19:18:17 +0800
comments: true
categories: java
---
Java中final的作用主要表现在三方面：修饰变量、修饰方法和修饰类。下面就从这两个方面来讲解final的作用。在文末从final及类的设计安全性出发，论述了Java中String为何要被设计成不可变类。  

1.final修饰变量  

final修饰变量时与C++中的const作用类似，即如果是基本类型变量，则表示其值不能改变；如果是引用类型变量，则表示其一旦初始化，就不能再指向别的对象，但是注意它指向的对象本身的值可以改变(其实这一点也跟C++中的常指针很像）。看下面一个例子<!--more-->即知。  

	import java.util.*;

	class Apple
	{
	  private float weight;
	  public Apple(float weight)
	  {
	    setWeight(weight);
	  }
	  public void setWeight(float weight)
	  {
	    this.weight=weight;
	  }
	  public void print()
	  {
	    System.out.println("Weight is "+String.valueOf(weight));
	  }
	  public void printHashCode()
	  {
	    System.out.println("HashCode:"+String.valueOf(hashCode()));
	  }
	}

	public class FinalSample
	{
	  public static void main(String[]args)
	  {
	    final int a=10;
	    //对于final修饰的基本类型变量，一旦初始化后就不能再被修改
	    //a=20;
	    final Apple apple=new Apple(300f);
	    apple.print();
	    apple.printHashCode();
	    //对于final修饰的引用类型变量，只是其指向不能变，但是其指向的对象本身可以改变。
	    apple.setWeight(350f);
	    apple.print();
	    apple.printHashCode();
	  }
	}

输出结果如下图所示：  

![java_final01](http://7xn1yt.com1.z0.glb.clouddn.com/java_final01.png)

显然，final修饰的基本类型变量a是不能被改变；由输出的哈希码不变可知apple始终指向同一个对象，但是它指向的这个对象的成员weight却发生了改变。  

也正是由于final对于引用类型变量“只能保证指向不变，却不能保证指向的对象本身不变”，所以在构造类时要特别小心，否则很容易被黑客利用。看下面一个例子即知。  

	import java.util.*;

	class AppleTag
	{
	  private float weight;
	  private float size;
	  public AppleTag(float weight,float size)
	  {
	    setWeight(weight);
	    setSize(size);
	  }
	  public void setWeight(float weight)
	  {
	    this.weight=weight;
	  }
	  public float getWeight()
	  {
	    return weight;
	   }
	  public void setSize(float size)
	  {
	    this.size=size;
	  }
	  public float getSize()
	  {
	    return size;
	  }
	}
	public class Apple
	{
	  private final AppleTag appleTag;
	  public Apple(AppleTag appleTag)
	  {
	    this.appleTag=appleTag;
	  }
	  public AppleTag getAppleTag()
	  {
	    return appleTag;
	  }
	  public void print()
	  {
	    System.out.println("Weight:"+String.valueOf(appleTag.getWeight())+
	                            " Size:"+String.valueOf(appleTag.getSize()));
	  }
	   
	  public static void main(String[]args)
	  {
	    AppleTag appleTag=new AppleTag(300f,10.3f);
	    Apple apple=new Apple(appleTag);
	    apple.print();
	   
	    appleTag.setWeight(13000f);
	    apple.print();
	   
	    AppleTag tag=apple.getAppleTag();
	    tag.setSize(100.3f);
	    apple.print();
	  }

	}

输出结果如下图所示：  

![java_final02](http://7xn1yt.com1.z0.glb.clouddn.com/java_final02.png)

类的设计者本意是想使appleTag一旦被初始化即不被修改，并且刻意不提供set函数以防止其被篡改。但实际上类的使用者却可以从两个地方修改appleTag从而达到改变apple的目的，其中一个地方是利用构造器中的实参进行修改，另一个就是利用getAppleTag()函数的返回值对其进行修改。  

显然，本例是由于类的设计不当而释放出过大的权限，使类的使用者将apple的重量和大小修改成了极不合理的值，从而得到错误的结果。那要如何避免这种错误呢？  

很简单，就是让final变量与外界充分隔离开，如本例中使类的使用者能获取相应的值，但是无法获取appleTag，代码如下所示：  

	import java.util.*;

	class AppleTag
	{
	  private float weight;
	  private float size;
	  public AppleTag(float weight,float size)
	  {
	    setWeight(weight);
	    setSize(size);
	  }
	  public void setWeight(float weight)
	  {
	    this.weight=weight;
	  }
	  public float getWeight()
	  {
	    return weight;
	   }
	  public void setSize(float size)
	  {
	    this.size=size;
	  }
	  public float getSize()
	  {
	    return size;
	  }
	}
	public class Apple
	{
	  private final AppleTag appleTag;
	  public Apple(AppleTag appleTag)
	  {
	    this.appleTag=new AppleTag(appleTag.getWeight(),appleTag.getSize());
	  }
	  public AppleTag getAppleTag()
	  {
	    return new AppleTag(appleTag.getWeight(),appleTag.getSize());
	  }
	  public void print()
	  {
	    System.out.println("Weight:"+String.valueOf(appleTag.getWeight())+
	                            " Size:"+String.valueOf(appleTag.getSize()));
	  }
	   
	  public static void main(String[]args)
	  {
	    AppleTag appleTag=new AppleTag(300f,10.3f);
	    Apple apple=new Apple(appleTag);
	    apple.print();
	   
	    appleTag.setWeight(13000f);
	    apple.print();
	   
	    AppleTag tag=apple.getAppleTag();
	    tag.setSize(100.3f);
	    apple.print();
	  }

	}

程序输出结果如下：  

![java_final03](http://7xn1yt.com1.z0.glb.clouddn.com/java_final03.png)

显然，尽管此处类的使用者仍然尝试越权去修改appleTag，却没有获得成功，原因就在于：在接收时，在类的内部新建了一个对象并让appleTag指向它；在输出时，使用者获取的是新建的对象，保证了使用者即可以获得相应的值，又无法利用获取的对象来修改appleTag。所以apple始终未被改变，三次输出结果相同。  

在我的博客《深刻理解Java中的String、StringBuffer和StringBuilder的区别》一文中已经说过String的对象是不可变的，因而在使用String时即使是直接返回引用也不用担心使用者篡改对象的问题，如下例：  

	import java.util.*;
	class Apple
	{
	  final String type;
	  public Apple(String type)
	  {
	    this.type=type;
	  }
	  public String getType()
	  {
	    return type;
	   }
	  public void print()
	  {
	    System.out.println("Type is "+type);
	   }
	}

	public class AppleTest
	{
	  public static void main(String[]args)
	  {
	    String appleType=new String("Red Apple");
	    Apple apple=new Apple(appleType);
	    apple.print();

	    appleType="Green Apple";
	    apple.print();

	    String type=apple.getType();
	    type="Yellow Apple";
	    apple.print();
	  }
	}

输出结果如下所示：  

![java_final04](http://7xn1yt.com1.z0.glb.clouddn.com/java_final04.png)

**显然，与前两个例子类似，类的使用者也尝试用相似的方法去修改type从而达到修改apple的目的，但是输出结果表明这种尝试没有成功。原因就在于String的对象不可变，在外面的修改实际上是让appleType及type分别指向了不同的对象，而用于初始化的String对象始终没有改变，当然apple中的type也就不改变了。  

再回过头来，我们发现曾经让我们很不爽的String(在深刻理解Java中的String、StringBuffer和StringBuilder的区别一文中讲到过，由于String对象不可变从而导致即使传引用也无法使对象的值改变），其实是经过Java设计者精心设计的，试想一下，如果String对象可变的话，在我们平常的程序编写中将会带来极大的安全隐患，而如果想杜绝这种隐患，则每次在使用String时都要经过精密考虑，程序也会变得更复杂，而偏偏String是被大量使用的（实际上不管哪种编程语言，字符串及其处理都占有相当大的比重），显然会给我们日常的程序编写带来极大的不便。  

另外一个值得注意的地方就是数组变量名其实也是一个引用，所以final修饰数组时，数组成员依然可以被改变。**  
