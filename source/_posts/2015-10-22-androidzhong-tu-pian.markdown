---
layout: post
title: "Android中图片,拍照及相册全面总结(1):图片压缩"
date: 2015-10-22 00:45:56 +0800
comments: true
categories: Android
---
由于现在手机像素越来越高，800万像素的手机，拍出来的照片就有2M左右,这么大的图片，如果用来直接显示的话，只要多几张，很容易就会发生OOM,但是用Thumbnail的话有不够清晰，所以需要对图片进行压缩之后在进行上传或显示。  

对图片压缩并上传分三个部分:  
第一,计算图片的缩放值<!--more-->,就是计算BitmapFactory.Options中的inSampleSize值,inSampleSize是用来控制缩放的.代码如下:  
	
	public static int calculateInSample(BitmapFactory.Options options,int reqWidth,int reqHeight)
	{
		final int height=options.outHeight;
		final int width=options.outWidth;
		int inSampleSize=1;
        if(height>reqHeight||width>reqWidth)
        {
        	final int heightRatio=Math.round((float)height/(float)reqHeight);
        	final int widthRatio=Math.round((float)width/(float)reqWidth);
        	inSampleSize=heightRatio<widthRatio?heightRatio:widthRatio;
        }
        
        return inSampleSize;
	}

问题是options.outHeight和options.outWidth是怎么来的呢？  
其实很简单，只要将BitmapFactory.Options的inJustDecodeBounds设置为true，就可以不把图片加载到内存中，也能测量出其大小。代码如下：  
	
	BitmapFactory.Options options=new BitmapFactory.Options();
	options.inJustDecodeBounds=true;
	BitmapFactory.decodeFile(filePath,options);
	int height=options.outHeight;
	int width=options.outWidth;


第二，decode获得bitmap,根据上面的讨论，现在可以根据图片路径获得图片并进行缩放;  
    
    //其中targetWidth,targetHeight是要缩放到的尺寸,比如450*300或者480*800这样
	public static Bitmap getScaledBitmap(String filePath,int targetWidth,int targetHeight)
	{
		final BitmapFactory.Options options=new BitmapFactory.Options();
        options.inJustDecodeBounds=true;
        BitmapFactory.decodeFile(filePath,options);

        options.inSampleSize=calculateInSampleSize(options,targetWidth,targetHeight);

        //then we need to set inJustDecodeBounds as false to decode file
        options.inJustDecodeBounds=false;
        
        return BitmapFactory.decodeFile(filePath,options);
	}

如果只是显示的话，到这一步就够了，但是往往还要上传,这就是第三步要讨论的问题;  

第三,指定压缩质量，然后按某种编码方式将其编码成String之后上传,这里以常用的JPEG压缩以及Base64编码方式为例;  

    //compressQuality表示压缩质量,一般取在30-100之间
	public String bitmap2String(String filePath,int compressQuality)
	{
		Bitmap bmp=getScaledBitmap(filePath,450,300);
		ByteArrayOutputStream baos=new ByteArrayOutputStream();
        bmp.compress(Bitmap.CompressFormat.JPEG,compressQuality,baos);
        byte[]b=baos.toByteArray();
        return Base64.encodeToString(b,Base64.DEFAULT);
	}

