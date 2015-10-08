---
layout: post
title: "Android-Universal-Image-Loader完全解析及改进(01)"
date: 2015-09-28 11:31:59 +0800
comments: true
categories: Android
---
 Android-Universal-Image-Loader(下面简称UIL)是由nostra13团队推出的一个的Android开源图片缓存库，项目地址为https://github.com/nostra13/Android-Universal-Image-Loader

 由于其良好的扩展性，以及较快的响应速度，受到了开发者的欢迎。
 
 从本文开始，将从UIL功能介绍，UIL整体架构分析，UIL重点类分析，UIL不足，UIL改进这五个方面进行介绍。本文先进行前面第一点的分析。 <!--more-->


1.UIL功能介绍  
1.1 UIL完成什么工作  
    简单的说，UIL就是根据用户的配置及要求，生成相应的请求并放入队列中，在后台通过线程池来处理队列中的请求，处理完成之后就将结果传递到UI线程（如果是同步加载则不传递到UI线程）,通知监听器，则UIL的工作完成。示意图如下所示：

![UIL work flow](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_whole_flow.png)

要注意的是上图只是描绘了图片缓存库的实现思想，而非UIL的具体实现，实际上，UIL的大致实现如下图，显然名称等跟上图有出入。

![UIL special flow](http://7xn1yt.com1.z0.glb.clouddn.com/UIL_architite_02.png)


1.2 UIL如何使用  
1.2.1 UIL配置  
   UIL的配置一般在Application或者Activity中完成。如下是一个典型的配置。  

   	public class UILApplication extends Application {
	@TargetApi(Build.VERSION_CODES.GINGERBREAD)
	@SuppressWarnings("unused")
	@Override
	public void onCreate() {
		if (Constants.Config.DEVELOPER_MODE && Build.VERSION.SDK_INT >= Build.VERSION_CODES.GINGERBREAD) {
			StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().detectAll().penaltyDialog().build());
			StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder().detectAll().penaltyDeath().build());
		}
		super.onCreate();
		initImageLoader(getApplicationContext());
	}

	public static void initImageLoader(Context context) {
		// This configuration tuning is custom. You can tune every option, you may tune some of them,
		// or you can create default configuration by
		//  ImageLoaderConfiguration.createDefault(this);
		// method.
		ImageLoaderConfiguration.Builder config = new ImageLoaderConfiguration.Builder(context);
		config.threadPriority(Thread.NORM_PRIORITY - 2);
		config.denyCacheImageMultipleSizesInMemory();
		config.diskCacheFileNameGenerator(new Md5FileNameGenerator());
		config.diskCacheSize(50 * 1024 * 1024); // 50 MiB
		config.tasksProcessingOrder(QueueProcessingType.LIFO);
		config.writeDebugLogs(); // Remove for release app

		// Initialize ImageLoader with configuration.
		ImageLoader.getInstance().init(config.build());
		}
	}	  		

1.2.2 DisplayImageOptions设置  
    DisplayImageOptions一般是在某个控件或某个Adapter（如ListView,GridView的Adapter)加载图片之前进行设置,包含占位图片，缓存设置，显示效果等。如下是一个ListView的Adapter中DisplayImageOptions的设置：  

	private static class ImageAdapter extends BaseAdapter {

		private static final String[] IMAGE_URLS = Constants.IMAGES;

		private LayoutInflater inflater;
		private ImageLoadingListener animateFirstListener = new AnimateFirstDisplayListener();

		private DisplayImageOptions options;

		ImageAdapter(Context context) {
			inflater = LayoutInflater.from(context);

			options = new DisplayImageOptions.Builder()
					.showImageOnLoading(R.drawable.ic_stub)
					.showImageForEmptyUri(R.drawable.ic_empty)
					.showImageOnFail(R.drawable.ic_error)
					.cacheInMemory(true)
					.cacheOnDisk(true)
					.considerExifParams(true)
					.displayer(new RoundedBitmapDisplayer(20)).build();
		}
		...
	}
        
1.2.3 调用ImageLoader的接口  
    如下是ImageLoader的主要pulic method如下:
    ![ImageLoader_apis](http://7xn1yt.com1.z0.glb.clouddn.com/ImageLoader_apis.png)
    其中init(ImageLoaderConfiguration)已经在1.2.1的配置中说过，其他的public method虽然多，但是很明显的可以看出分为displayImage(...),loadImage(...),loadImageSync(...)这三类。
    其中最常用的就是displayImage(...),如果要在控件上显示图片一般都使用displayImage(...);
    如果不是要马上在控件上显示，则可考虑使用loadImage(...);
    如果要同步加载，则使用loadImageSync(...);  
    
    如下Adapter中的一个典型使用:  
	
	@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		ImageView imageView = (ImageView) convertView;
		if (imageView == null) {
			imageView = (ImageView) inflater.inflate(R.layout.item_gallery_image, parent, false);
		}
		ImageLoader.getInstance().displayImage(IMAGE_URLS[position], imageView, options);
		return imageView;
	}  
1.2.4 注意事项  
    由于ImageLoader需要从网络获取图片并进行memory cache以及disk cache，故要在Manifest文件中加上以下权限：  

	<uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />    

2.UIL整体架构分析  
   
  见 [Android-Universal-Image-Loader完全解析及改进(02)](http://blog.imallen.wang/blog/2015/09/30/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-02/) 

3.UIL重点类分析  

 见 [Android-Universal-Image-Loader完全解析及改进(02)](http://blog.imallen.wang/blog/2015/09/30/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-02/) 以及 [Android-Universal-Image-Loader完全解析及改进(03)](http://blog.imallen.wang/blog/2015/10/01/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-03/)  

4.UIL不足  

 见[Android-Universal-Image-Loader完全解析及改进(04)](http://blog.imallen.wang/blog/2015/10/01/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-04/)

5.UIL改进  
 见[Android-Universal-Image-Loader完全解析及改进(04)](http://blog.imallen.wang/blog/2015/10/01/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-04/)

