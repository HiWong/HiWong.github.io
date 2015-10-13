---
layout: post
title: "深入分析Java中的I/O类的特征及适用场合"
date: 2015-10-13 15:04:36 +0800
comments: true
categories: Java
---
Java中有40多个与输入输出有关的类，如果不理清它们之间的关系，就不能灵活地运用它们。  

如果从流的流向来分，可分为输入流和输出流，而输入流和输出流又都可分为字节流和字符流。因而可将Java中的I/O流以下图进行划分：  

![javaio_01](http://7xn1yt.com1.z0.glb.clouddn.com/javaio_01.png)

注意上图并非继承关系，而只是一个示意<!--more-->图。  

Java中的其它与I/O流处理相关的类其实都是从InputStream,Reader,OutputStream和Writer这4个基类继承而来。其中InputStream和OutputStream为字节流，Reader和Writer为字符流。  

之所以这样划分，是因为计算机中所有的数据都是二进制的，而字节流可处理所有的二进制文件，但如果使用字节流来处理文本文件，也不是不可以，但是会更麻烦，所以通常有如下一个规则：如果进行输入/输出的内容是文本文件，则应该考虑使用字符流；如果进行输入/输出的内容是二进制内容(如图片、音频文件），则应该考虑使用字节流。  

下面是从InputStream,Reader,OutputStream,Writer这4个基类出发，列出了常用的子类:  

![javaio_02](http://7xn1yt.com1.z0.glb.clouddn.com/javaio_02.png)  

![javaio_03](http://7xn1yt.com1.z0.glb.clouddn.com/javaio_03.png)  

![javaio_04](http://7xn1yt.com1.z0.glb.clouddn.com/javaio_04.png)  

![javaio_05](http://7xn1yt.com1.z0.glb.clouddn.com/javaio_05.png)

下面对各个类的特点及适用场合进行说明：  

1.其中InputStreamReader和OutputStreamWriter是比较特殊的类，它们可将字节流转换为字符流，因而称为转换流。如InputStreamReader reader=new InputStreamReader(System.in)；  

2.BufferedInputStream,BufferedOutputStream的用法虽然和FileInputStream,FileOutputStream的用法一样，但是效率却相差很大，因为内存的效率比IO操作的效率要高得多，而BufferedInputStream,BufferedOutputStream会提前将文件中的内容读取(写入)到缓冲区，当读取(写入)时如果缓冲区存在就直接从缓冲区读，只有缓冲区不存在相应内容时才会读取(写入)新的数据到缓冲区，而且通常会请求的数据要多。所以在读取(写入)文件，特别是较大的文件时，不要用最简单的FileInputStream和FileOutputStream,而要考虑使用BufferedInputStream,BufferedOutputStream.  

3.ObjectInputStream,ObjectOutputStream则分别是将序列化的对象(即实现了Serializable接口的对象)读取出来/写入到文件中，显然，这其实是利用反射的原理。  

4.前面说过，Reader与InputStream的区别在于一个是字符输入流，一个是字节输入流。FileReader与FileInputStream对应，BufferedReader与BufferedInputStream对应，所以在读取文本文件时最好使用BufferedReader而不要使用FileReader.  

由于ByteArrayInputStream及PipedInputStream不常用，本文暂不讨论。  

下面是一些代码实例：  

首先是比较原始的读取文件的方法,即采用FileInputStream:  

	//这是读取字节流的实例
	public class FileInputStreamSample {

		public static void main(String[]args) throws IOException
		{
			FileInputStream fis=new FileInputStream("d://error.dat");
			byte[]buff=new byte[1024];
			int hasRead=0;
			while((hasRead=fis.read(buff))>0)
			{
				System.out.println(new String(buff,0,hasRead));
			}
	        
			fis.close();
		
		}
	}

然后是利用FileOutputStream进行文件的写入：  

	public class FileOutputStreamSample {

		public static void main(String[]args)
		{
			String fileName="d://error.dat";
			//注意：不必提前新建，因为如果没有新建的话，它自己会新建一个。
			String newFileName="d://error2.txt";
			try
			(FileInputStream fis=new FileInputStream(fileName);
			 FileOutputStream fos=new FileOutputStream(newFileName))
			 {
				byte[]buff=new byte[32];
				int hasRead=0;
				while((hasRead=fis.read(buff))>0)
				{
					fos.write(buff,0,hasRead);
				}
			 }
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}
	}

比较高效的读取文件的方法，即采用BufferedInputStream:  

	public class BufferedInputStreamSample {

		public static void main(String[]args)
		{
			File file=new File("d://error.dat");
			try
			(BufferedInputStream bis=new BufferedInputStream(new FileInputStream(file));)
			{
			    byte[]buff=new byte[1024];
			    int hasRead=0;
			    while((hasRead=bis.read(buff))>0)
			    {
			    	String content=new String(buff,0,hasRead);
			    	System.out.println(content);
			    }	
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
			
		}
	}

下面是FileReader的用法，但是注意FileReader读取文本文件是比较低效的方法：   

	public class FileReaderSample {

		public static void main(String[]args)
		{
			try(
					FileReader fr=new FileReader("d://error.dat")
				)
			{
				char[]cbuff=new char[32];
				int hasRead=0;
				while((hasRead=fr.read(cbuff))>0)
				{
					System.out.println(new String(cbuff,0,hasRead));
				}
				
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}
	}

类似地，FileWriter是比较低效的写入文本文件的方法：  

	public class FileWriterSample {

		public static void main(String[]args)
		{
			try
			(FileWriter fw=new FileWriter("d://poem.txt"))
			{
				fw.write("I have a dream\n");
				fw.write("One day on the red hills of Georgia\n");
				fw.write("The sons of former slaves and the sons of  former slave owner will be able to sit down together at the table.\n");
				fw.write("My four little children will one day live in a nation where they will not be judged by the color of their skin but by the content of their character.\n");
				
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}
	}

而BufferedReader是较高效的读取文本文件的方式，但是注意它的构造方法需要一个InputStreamReader，而InpuStreamReader又是包装FileInputStream而来,所以BufferedReader的使用方法如下：  

	public class BufferedReaderSample {

		public static void main(String[]args)
		{
			try
			(
					//如果是读取文件，则为InputStreamReader reader=new InputStreamReader(new InputStream("d://error.dat"));
					//InputStreamReader reader=new InputStreamReader(new FileInputStream("d://error.dat"));
					InputStreamReader reader=new InputStreamReader(System.in);
					BufferedReader br=new BufferedReader(reader)			
			)
			{
			      String buffer=null;
			      while((buffer=br.readLine())!=null)
			      {
			    	  System.out.println(buffer.toUpperCase());
			      }
				
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}
	}

显然，BufferedReader的好处是具有缓冲功能，它可以一次读取一行文本----以换行符为标志，上例是读取键盘输入后转换为大写并输出，当然，也可以读取文件后将各行转换为大写后输出。
BufferedWriter用法较简单，但是值得注意的是它要flush:  

	public class BufferedWriterSample {

		public static void main(String[]args)
		{
			try
			(
			     //如果是写入到文件则为OutputStreamWriter writer=new OutputStreamWriter(new FileOutputStream("d://error.dat"));
			     OutputStreamWriter writer=new OutputStreamWriter(System.out);
				 BufferedWriter bw=new BufferedWriter(writer);
			)
			{
				String content="Less is more\nLess is more is not a law\nLess is more is not always correct";
				bw.write(content);	
				bw.flush();
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
			
		}
	}

下面是PrintStream,PrintWriter,BufferedWriter这3个类的对比：  

首先是它们的共同点：都是处理流(包装流）而非节点流，因而可以更方便地使用，如PrintStream使用println(String)功能，BufferedWriter使用writer(String)功能；  

PrintStream与PrintWriter,BufferedWriter的区别在于前者是处理字节流，而后两者是处理字符流；而BufferedWriter与PrintWriter相比，由于缓冲区的作用，它的效率要比PrintWriter要高。  

下面是一个PrintStream的例子：  

	class Student
	{
		int id;
		String name;
		public Student()
		{
			id=0;
			name="Jenny";
		}
		public String toString()
		{
			return "id="+id+" name="+name;
		}
	}
	public class PrintStreamSample {

		public static void main(String[]args)
		{
			String fileName="d://poem.txt";
			try
			(PrintStream ps=new PrintStream(new FileOutputStream(fileName)))
			{
	            //注意：这会把以前的覆盖，要想不覆盖的话，就要使用ps.append的方法而不是println的方法。
				ps.println("Less is more");
				//直接使用println输出对象，这个在Socket编程时很有用。
				ps.println(new Student());
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}
	}

下面是PrintWriter的一个例子：  

	public class PrintWriterSample {

		public static void main(String[]args)
		{
			try
			(
					PrintWriter writer=new PrintWriter(new OutputStreamWriter(System.out));				
			)
			{
				writer.println("Less is more is a important rule.");
				writer.println(true);		
			}
		 
		}
	}

最后是ObjectInputStream及ObjectOutputStream，利用这两个类来读写序列化对象特别方便，如下所示：  

	public class ObjectOutputStreamSample {

		public static void main(String[]args)
		{
			Student stu1=new Student(1,"Jack","NewYork");
			Student stu2=new Student(2,"Rose","California");
			
			File file=new File("d://object.txt");
			//由此可见，BufferedInputStream以及ObjectOutputStream其实都是对FileOutputStream进行了包装。
			try
			(ObjectOutputStream oos=new ObjectOutputStream(new FileOutputStream(file)))
			{
				oos.writeObject(stu1);
				oos.writeObject(stu2);
				
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
			
			
			
		}
	}
	class Student implements Serializable{

		private int id;
		private String name;
		private String address;
		
		public Student(int id,String name,String address)
		{
			this.id=id;
			this.name=name;
			this.address=address;
		}
		
		@Override
		public String toString()
		{
			StringBuilder sb=new StringBuilder();
			sb.append("id:"+id);
			sb.append("    ");
			sb.append("name:"+name);
			sb.append("    ");
			sb.append("address:"+address);
			return sb.toString();
		}
		
	}

	public class ObjectInputStreamSample {

		public static void main(String[]args)
		{
			File file=new File("d://object.txt");
			try
			(ObjectInputStream ois=new ObjectInputStream(new FileInputStream(file)))
			{
				Student stu1=(Student)ois.readObject();
				Student stu2=(Student)ois.readObject();
			}
			catch(Exception ex)
			{
				ex.printStackTrace();
			}
			
		}
    }

