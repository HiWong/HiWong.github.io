---
layout: post
title: "设计模式(3):建造者模式和原型模式"
date: 2016-02-16 01:29:35 +0800
comments: true
categories: design_pattern
---

1.建造者模式  
工厂类模式提供的是创建单个类的模式，而建造者模式则是将各种产品集中起来进行管理，用来创建复合对象。所谓复合对象就是指某个类具有不同的属性。建造者模式的实质就是将一个复杂对象与它的表示分离，使得同样的构建过程可以创建不同的表示。  

该模式的使用场景为：1)相同的方法，不同的执行顺序，产生不同的事件结果时；2)多个部件或零件，都可以装配到一个对象中，但是产生的运行结果又不相同时；3）产品类非常复杂，或者产品类的调用顺序不同产生了不同的效能，这个时候使用建造者模式非常合适，经典的应用就是ImageLoader和Android中的AlertDialog.  

建行者模式的UML图如下所示<!--more-->：  

![BuildPattern](http://7xn1yt.com1.z0.glb.clouddn.com/BuildPattern.png)

需要注意的是，Director并不是一定必须的，事实上，有许多使用建造者模式的场合没有使用Director.  

1.1 建造者模式的简单示例  

比如建房子需要建设墙壁、房梁、窗户，组装电脑需要先建造CPU、RAM、ROM等。下面以建设房子为例。代码如下所示：  

	public abstract class House{
		private int wall;
		private int beam;
		private int window;

		public House(){}

		public abstract void setWall(int wall);
		public abstract void setBeam(int beam);
		public abstract void setWindow(int window);

		@Override
		public String toString(){
			return "House is consist of "+wall+" walls, "+beam+" beams and "+window+" windows";
		}
	}

	public class TraditionHouse extends House{
		public TraditionHouse(){}

		@Override
		public void setWall(int wall){
			this.wall=wall;
		}

        @Override
        public void setBeam(int beam){
        	this.beam=beam;
        }

        @Override
        public void setWindow(int window){
        	this.window=window;
        }


	}

	public abstract class Builder{
		public abstract void buildWall(int wall);
		public abstract void buildBeam(int beam);
		public abstract void buildWindow(int window);

		public abstract House create();
	}

	public class TraditionHouseBuilder extends Builder{
		private House house=new TraditionHouse();

		@Override
		public void buildWall(int wall){
			house.setWall(wall);
		}

		@Override
		public void buildBeam(int beam){
			house.setBeam(beam);
		}

		@Override
		public void buildWindow(int window){
			house.setWindow(window);
		}

		@Override
		public House create(){
			return house;
		}

	}

	public class Director{
		Builder builder=null;

		public Director(Builder builder){
			this.builder=builder;
		}

		public void construct(int wall,int beam,int window){
			builder.buildWall(wall);
			builder.buildBeam(beam);
			builder.buildWindow(window);
		}

	}


测试代码如下:

	public class Test{
		public static void main(String[]args){
			Builder builder=new TraditionBuilder();
			Director director=new Director(builder);
			director.construct(8,12,20);
			System.out.println("House info:"+builder.create().toString());
		}
	}

1.2 Android源码中建造者模式的使用  
在Android源码中，有很多地方用到了建造者模式，其中很经典的一个例子就是AlertDialog.Builder，使用该Builder来构建复杂的AlertDialog对象，如下是一个使用AlertDialog的示例：  

	private void showAlertDialog(Context contet){
		AlertDialog.Builder builder=new AlertDialog.Builder(context);
		builder.setIcon(R.drawable.logo);
		builder.setTitle(ResUtils.getString(R.string.title));
		builder.setMessage(ResUtils.getString(R.string.msg));
		builder.setPositiveButton(ResUtils.getString(R.string.sure),
		 new DialogInterface.OnClickListener(){
			@Override
			public void onClick(DialogInterface dialog,int whichButton){
				//action code for positive button
			}
		});
		builder.setNeutralButton(ResUtils.getString(R.string.ok),
		  new DialogInterface.OnClickListener(){
		  	 @Override
		  	 public void onClick(DialogInterface dialog,int whichButton){
		  	 	//action code for neutral button
		  	 }
		  	});
		builder.setNegativeButton(ResUtils.getString(R.string.cancel),
		   new DialogInterface.OnClickListener(){
		   	  @Override
		   	  public void onClick(DialogInterface dialog,int whichButton){
		   	  	//action code for negative button
		   	  }
		   	});

		builder.create().show();

	}



2.原型模式  
原型模式虽然是创建型的模式，但是与工程模式没有关系，从名字即可看出，该模式的思想就是将一个对象作为原型，对其进行复制、克隆，产生一个和原对象类似的新对象。下面是一个简单的原型类:  

	public class Prototype implements Cloneable{
		public Object clone() throws CloneNotSupportedException{
			Prototype proto=(Prototype)super.clone();
			return proto;
		}
	}

很简单，一个原型类，只需要实现Cloneable接口，重写clone方法。但是其实Cloneable接口是个空接口，可以任意定义实现类的方法名，如cloneA或cloneB，因为此处的重点是super.clone()这句，super.clone()调用的是Object的clone()方法，而在Object类中，clone是native的，具体怎么实现，会在后面的博客中讲解，此处暂时不深究。  
学过C++的同学应该都知道浅复制和深复制的概念。  
浅复制:将一个对象复制后，基本数据类型的变量都会重新创建，而引用类型，指向的还是原对象所指向的，即实际上并没有重新创建对象，而只是复制了一个指针;
深复制：将一个对象复制后，不论是基本数据类型还是引用类型，都是重新创建的。简单来说，就是深复制进行了完全彻底的复制，而浅复制不彻底。  

下面，写一个深浅复制的例子：  

	public class Prototype implements Cloneable,Serializable{
		private static final long serialVersionUID=1L;
		private String string;

		private SerializableObject obj;

		//shallow clone
		public Object clone() throws CloneNotSupportedException{
			Prototype proto=(Prototype)super.clone();
			return proto;
		}

		//deep clone
		public Object deepClone() throws IOException,ClassNotFoundException{
			ByteArrayOutputStream bos=new ByteArrayOutputStream();
			ObjectOutputStream oos=new ObjectOutputStream(bos);
			oos.writeObject(this);

			ByteArrayInputStream bis=new ByteArrayInputStream(bos.toByteArray());
			ObjectInputStream ois=new ObjectInputStream(bis);
			return ois.readObject();
		}

		public String getString(){
			return string;
		}

		public void setString(String str){
			this.string=str;
		}

		public SerializableObject getObj(){
			return obj;
		}

		public void setObj(SerializableObject obj){
			this.obj=obj;
		}
	}

	class SerializableObject implements Serializable{
		private static final long serialVersionUID=1L;
	}