---
layout: post
title: "Android-Universal-Image-Loader完全解析及改进(04)"
date: 2015-10-01 22:09:07 +0800
comments: true
categories: Android
---
在 [Android-Universal-Image-Loader完全解析及改进(02)](http://blog.imallen.wang/blog/2015/09/30/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-02/) 以及 [Android-Universal-Image-Loader完全解析及改进(03)](http://blog.imallen.wang/blog/2015/10/01/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-03/) 这两篇博客中，分析了UIL的整体架构和重点类，同时也零星地指出了UIL存在的一些不足。  
本文将罗列出UIL存在的不足并给出相应的建议，并且在文章最后将给出一个我自己写的改进的图片加载框架。<!--more-->  

4.1 IoUtils中的copyStream(...)方法中，应该使用BufferedInputStream和BufferedOutputStream，因为内存操作的效率比IO操作的效率要高得多，BufferedInputStream,BufferedOutputStream会提前将文件中的内容读取(写入)到缓冲区，当读取(写入)时如果缓冲区存在就直接从缓冲区读，只有缓冲区不存在相应内容时才会读取(写入)新的数据到缓冲区，而且通常会请求的数据要多。  

4.2 LoadAndDisplayImageTask中delayIfNeed()使用了Thread.sleep(options.getDelayBeforeLoading());其实是一种十分不优雅的方式,但是如果使用wait()...notify()模式的话，则需要在ImageLoaderEngine中维护一份Monitor表，成本比较高，所以比较好的方式是在ImageLoader或ImageLoaderEngine中利用handler.postDelayed(Runnable,long)实现延时操作。  

4.3 BaseMemoryCache#public Collection<String>keys()中，synchronized是多余的，因为softMap本身就是线程安全的。  

4.4 FuzzyMemoryCache,LruDiskCache都存在内聚性低，耦合性高的缺点。  
另外，FuzzyMemoryCache中虽然实现了不会重复插入，但是却没有实现keyComparator判定为相等的key能获取一样的value。综合考虑以上两点，下面是一个较好的实现。  

	public class FuzzyKeyMemoryCache implements MemoryCache{

	    private static final String TAG=FuzzyKeyMemoryCache.class.getSimpleName();

	    private final MemoryCache cache;

	    public FuzzyKeyMemoryCache(MemoryCache memoryCache)
	    {
	        this.cache=memoryCache;
	    }

	    /**
     	- we should use imageUri as key
     	- @param key
     	- @param value
     	- @return
	     */
	    @Override
	    public boolean put(String key,Bitmap value)
	    {	        
	        return cache.put(MemoryCacheUtils.getImageUri(key),value);
	    }

	    @Override
	    public Bitmap get(String key)
	    {
	        return cache.get(MemoryCacheUtils.getImageUri(key));
	    }

	    @Override
	    public Bitmap remove(String key)
	    {
	        return cache.remove(MemoryCacheUtils.getImageUri(key));
	    }

	    @Override
	    public void clear()
	    {
	        cache.clear();
	    }

	    @Override
	    public Collection<String>keys()
	    {
	        return cache.keys();
	    }

    }

4.5 ImageLoader中，private volatile static ImageLoader instance;中的volatile是多余的;  

4.6 在LoadAndDisplayImageTask中有以下代码：  

	private boolean waitIfPaused()
    {
        AtomicBoolean pause=engine.getPause();
        if(pause.get())
        {
            synchronized (engine.getPauseLock())
            { 
                if(pause.get())
                {
                    L.d(LOG_WAITING_FOR_RESUME, memoryCacheKey);
                    try
                    {
                        engine.getPauseLock().wait();
                    }
                    catch(InterruptedException ex)
                    {
                        L.e(LOG_TASK_INTERRUPTED, memoryCacheKey);
                        return true;
                    }
                    L.d(LOG_RESUME_AFTER_PAUSE, memoryCacheKey);
                }
            }


        }
        return isTaskNotActual();
    }

其中的if(pause.get())应该改为while(pause.get())，否则在wait过程中可能出现假醒。  

4.7 ImageLoader#defineHandler(...)方法如下：  

	private static Handler defineHandler(DisplayImageOptions options) {
		Handler handler = options.getHandler();
		if (options.isSyncLoading()) {
			handler = null;
		} else if (handler == null && Looper.myLooper() == Looper.getMainLooper()) {
			handler = new Handler();
		}
		return handler;
	}

显然只要用户没有给DisplayImageOptions中的handler赋值，就会每次都新建一个Handler，其实可以只在第一次需要的时候新建一次，然后一直复用。  

**5 总结**   
  **我自己在看UIL代码的过程中，发现虽然总体的架构没什么问题，但是有不少像上面列举的问题，还有一些地方可以采用更好的实现，比如ImageDownloader缺少一个内聚性高的包装类。所以在UIL的基础上我写了一个改进的ImageLoader,为了与UIL区分开,加入了我的英文名,取名为AllenImageLoader,感兴趣的童鞋可以到我的github上fork：[AllenImageLoader](https://github.com/HiWong/AllenImageLoader)  
  喜欢的话记得给小星星哦!**


