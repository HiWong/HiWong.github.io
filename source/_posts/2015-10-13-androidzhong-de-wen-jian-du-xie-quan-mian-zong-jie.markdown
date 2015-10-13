---
layout: post
title: "Android中的文件读写全面总结"
date: 2015-10-13 14:51:46 +0800
comments: true
categories: Android
---
在深入分析Java中的I/O类的特征及适用场合 一文中，我详细介绍了Java中的I/O，但是，如果以为Android中的I/O与Java中一样，那就大错特错了。实际上，它们有一定的相同之外，但更多的是区别，因为Android系统中的文件存放位置不同，读取方式也不一样。下面将详细介绍Android中的文件读写：  

一、资源文件的读取，不需要在Manifest文件中添加权限  

1.从resource中的asset中读取文件，要注意的是asset中的文件只能读<!--more-->而不能写  

	public static void readFromAsset(Context context,String fileName)
		{
			try
			{
			    InputStream in=context.getResources().getAssets().open(fileName);
			    int length=in.available();
			    byte[]buffer=new byte[length];
			    
			    in.read(buffer);
			    in.close();
			    String content=new String(buffer,"UTF-8");
			    ToastUtils.showShortToast(context, content);    
			
			}
		    catch(IOException e)
		    {
		    	e.printStackTrace();
		    }
				
		}

2.从resource的raw中读取文件，与上面的一样，也是只能读取而不能写入  


	public static void readFromRaw(Context context,int rawResId)
		{
		      InputStream in=context.getResources().openRawResource(rawResId);
		      try
		      {
		    	  int length=in.available();
			      byte[]buffer=new byte[length];
			      in.read(buffer);
			      
			      String res=EncodingUtils.getString(buffer, "UTF-8");
			      
		      }
		     catch(IOException ex)
		     {
		    	 ex.printStackTrace();
		     }
		     finally
		     {
		    	 if(in!=null)
		    	 {
		    		 try
		    		 {
		    			 in.close();
		    		 }
		    		 catch(Exception e)
		    		 {
		    			 e.printStackTrace();
		    		 }
		    	 }
		     }
		}

看到上面两段示例代码可能有人会问，会什么不想Java中那样使用try(InputStream in=context.getResources().getAssets().open(fileName))这种autoclose的方式，但是实际上它需要Java SE1.7的版本才支持，而目前的Android版本并不支持，因而无法使用这种方法。  
二、读写应用包名目录(即/data/data/packagename)下的文件  

首先是读文件  

	public static void readFromPackage(Context context,String fileName)
		{
			try
			{
				FileInputStream fis=context.openFileInput(fileName);
				int length=fis.available();
				byte[]buffer=new byte[length];
				fis.read(buffer);
				
				String content=new String(buffer,"UTF-8");
				ToastUtils.showLongToast(context, content);
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}
	
然后是写文件，注意权限最好是Context.MODE_PRIVATE，否则有时候会触犯安全红线。目前的模式主要有以下几种：  
Context.MODE_PRIVATE:为默认操作模式，代表该文件是私有数据，只能被应用本身访问，在该模式下，写入的内容会覆盖原文件的内容，如果想把新写入的内容追加到原文件中，可以使用Context.MODE_APPEND;  

Context.MODE_APPEND:该模式会检查文件是否存在，若存在就往文件追加内容，否则就创建新文件；  

Context.MODE_WORLD_READABLE：表示当前文件可以被其他应用读取；  

Context.WORLD_WRITEABLE:表示当前文件可以被其他应用写入；  

	public static void writeToPackage(Context context,String fileName,String str)
		{
			try
			{
				FileOutputStream fos=context.openFileOutput(fileName,Context.MODE_PRIVATE);
				byte[]buffer=str.getBytes();
				
				fos.write(buffer);
				fos.close();
				
			}
			catch(IOException ex)
			{
				ex.printStackTrace();
			}
		}

三、读写sd卡中的文件，注意要添加以下权限：  

	<!--写入数据到SD卡-->
	<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>

	<!--创建或删除SD卡中的文件-->
	<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>

首先是从SD卡读取文件  

     public void readFromSdcard(Context context,String fileName)
     {
    	 try
    	 {
    		 FileInputStream fis=new FileInputStream(fileName);
    		 int length=fis.available();
    		 byte[]buffer=new byte[length];
    		 fis.read(buffer);
    		 
    		 String content=new String(buffer,"UTF-8");
    		 ToastUtils.showLongToast(context, content);		 
    	 }
    	 catch(IOException ex)
    	 {
    		 ex.printStackTrace();
    	 }
     }

然后是写入文件到SD卡：  

	public static void writeToSdCardFile(byte[]buffer,Context context,String fileName)
     {
    	 try
    	 {
    		 FileOutputStream fos=new FileOutputStream(fileName);
    		 fos.write(buffer);
    		 fos.close();
    	 }
    	 catch(IOException ex)
    	 {
    		 ex.printStackTrace();
    	 }
    	 
     }

当然，上面只是进行了最基础的示例，至于获取到文件流之后的包装问题（如采用缓冲流等），就深入分析Java中的I/O类的特征及适用场合 一文中介绍得完全一样了，也建议在实际使用时不要采用最基本的文件操作流，由于在之前的文章中详细讲过，这里就不展开讨论了。  
