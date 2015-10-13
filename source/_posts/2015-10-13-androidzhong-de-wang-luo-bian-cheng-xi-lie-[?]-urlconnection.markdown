---
layout: post
title: "Android中的网络编程系列(一):URLConnection"
date: 2015-10-13 13:26:28 +0800
comments: true
categories: android
---
URL(Uniform Resource Locator)对象代表统一资源定位器，它是指向互联网资源的指针。URL由协议名、主机、端口和资源路径组件，即满足如下格式：  

	protocol://host:port/path

例如http://kan.sogou.com/dianying/就是一个URL地址。  

URL提供了多个构造方法用于创建URL对象，同时它提供的主要方法如下：  
(1)String getFile():获取此URL的资源名；  

(2)String getHost():获取此URL的主机名；  

(3)String getPath():获取此URL的路径<!--more-->部分；  
 
(4)String getProtocol():获取此URL的协议名称；  

(5)String getQuery():获取此URL的查询字符串部分；  

(6)URLConnection  
openConnection():返回一个URLConnection对象，它表示URL所引用的远程对象与本进程的连接；  

(7)InputStream openStream():打开与此URL的连接，并返回一个用于读取该URL资源的InputStream;   
下面以下载Baidu上的一张死神的图片为例，示范如何使用URL:  

layout文件很简单，就两个button和一个ImageView，就不贴上来了，下面是源码：	  

	public class MainActivity extends Activity implements View.OnClickListener{
		private static final String PIC_URL="http://pic7.nipic.com/20100528/4981527_163140644317_2.jpg";
		private static final int SHOW_IMAGE=0x123;
		private static final int SAVE_SUCC=0x124;
		private static final int SAVE_ERROR=0x125;
		
		Button showImageButton,saveImageButton;
		ImageView imageView;
		Bitmap bitmap;
		
		private Handler handler=new Handler()
		{
			@Override
			public void handleMessage(Message msg)
			{
				if(msg.what==SHOW_IMAGE)
				{
					imageView.setImageBitmap(bitmap);
				}
				else if(msg.what==SAVE_SUCC)
				{
					Toast.makeText(getBaseContext(), "图片保存成功", Toast.LENGTH_LONG).show();
				}
				else if(msg.what==SAVE_ERROR)
				{
					Toast.makeText(getBaseContext(), "图片保存出错", Toast.LENGTH_LONG).show();
				}
			}
		};
		
		private void initView()
		{
			showImageButton=(Button)findViewById(R.id.showImageButton);
			saveImageButton=(Button)findViewById(R.id.saveImageButton);
			imageView=(ImageView)findViewById(R.id.imageView);
		}
		
		@Override
		protected void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			
			initView();
			showImageButton.setOnClickListener(this);
			saveImageButton.setOnClickListener(this);
		}

		@Override
		public boolean onCreateOptionsMenu(Menu menu) {
			// Inflate the menu; this adds items to the action bar if it is present.
			getMenuInflater().inflate(R.menu.main, menu);
			return true;
		}

		private void showImage()
		{
			new Thread()
			{
				@Override
				public void run()
				{
					try 
					{
						URL url = new URL(PIC_URL);
						InputStream is=url.openStream();
						bitmap=BitmapFactory.decodeStream(is);
						handler.sendEmptyMessage(SHOW_IMAGE);
			            is.close();
					} 
					catch(Exception e) 
					{
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
				}
			}.start();
			
		}
		
		public void saveImage()
		{
			new Thread()
			{
				@Override
				public void run()
				{
					try		
					{
						URL url=new URL(PIC_URL);
						InputStream is=url.openStream();
						BufferedInputStream bis=new BufferedInputStream(is);
						
						String filePath="animation.png";
						FileOutputStream fos=getBaseContext().openFileOutput(filePath,MODE_PRIVATE);
						BufferedOutputStream bos=new BufferedOutputStream(fos);
						
						byte[]buff=new byte[32];
						int hasRead=0;
						
						while((hasRead=bis.read(buff))>0)
						{
							bos.write(buff,0,hasRead);
						}
						
						
						bos.close();
						fos.close();
						
						bis.close();
						is.close();
						
					    handler.sendEmptyMessage(SAVE_SUCC);
					}
					catch(Exception ex)
					{
						handler.sendEmptyMessage(SAVE_ERROR);
						ex.printStackTrace();
					}
				}
			}.start();
		}
		
		@Override
		public void onClick(View v) {
			// TODO Auto-generated method stub
			switch(v.getId())
			{
			case R.id.showImageButton:
				showImage();
				break;
			case R.id.saveImageButton:
				saveImage();
				break;
			}
		}

	}

运行结果如下：  

![android_urlconnection](http://7xn1yt.com1.z0.glb.clouddn.com/Android_urlconnection.png)
