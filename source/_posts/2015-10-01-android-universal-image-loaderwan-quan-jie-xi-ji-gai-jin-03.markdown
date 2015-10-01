---
layout: post
title: "Android-Universal-Image-Loader完全解析及改进(03)"
date: 2015-10-01 12:56:13 +0800
comments: true
categories: 
---
在 [Android-Universal-Image-Loader完全解析及改进(02)](http://blog.imallen.wang/blog/2015/09/30/android-universal-image-loaderwan-quan-jie-xi-ji-gai-jin-02/) 一文中分析了UIL的整体架构，实现流程已经最核心的几个类。本文将分析剩下的重点类。  

3.1 MemoryCache家族  
MemoryCache家族包括BaseMemoryCache,LimitedMemoryCache,LargestLimitedMemoryCache,  
LRULimitedMemoryCache,LruMemoryCache,LimitedAgeMemoryCache,FuzzyKeyMemoryCache等.   
之所以叫MemoryCache家族是因为其中的类都是全部或部分实现了MemoryCache接口。值得注意的是这里充分利用了"组合优于继承"这一点,如FuzzyKeyMemoryCache。另外，从接口到抽象类，再到完全实现类，是一种经典的设计方法。  
<!--more-->
3.1.1 BaseMemoryCache  
实现了MemoryCache基本功能的抽象类，由于采用Reference<Bitmap>,有利于虚拟机在内存不足时回收缓存对象。其抽象函数为:

	protected abstract Reference<Bitmap> createReference(Bitmap value);    

显然，子类在实现该方法时，可以根据需要采用SoftReference或WeakReference.
比如WeakMemoryCache就采用了WeakReference<Bitmap>,从而实现弱引用缓存。

3.1.2 LimitedMemoryCache及其衍生类  
限制缓存总大小，并继承自BaseMemoryCache的抽象类。注意它并没有实现BaseMemoryCache中的createReference(Bitmap)这个抽象方法。  
每次put时，会先判断大小是否超出上限，若是则一直删除对象到小于上限。删除法则由抽象方法removeNext()决定。另外，抽象方法getSize(Bitmap)表示一个元素的大小。LimitedMemory的衍生类如下。  

1)LRULimitedMemoryCache  
限制总大小的内存缓存,在达到上限时会优先删除最近最少使用的元素(其实就是LRU算法)。  
其实现方法是通过构造new LinkedHashMap<String,Bitmap>(INITIAL_CAPACITY,LOAD_FACTOR,true),其中true表示根据accessOrder进行排序，使用次数越多的越排在后面，所以每次删除时只要将第一个元素删除即可。  

2)UsingFreqLimitedMemoryCache  
限制总大小的内存缓存，会在缓存满时优先删除总使用次数最少的元素。  

3)LargestLimitedMemoryCache  
限制总大小的内存缓存,会在缓存满时优先删除最大的元素。  

4)FIFOLimitedMemoryCache  
限制总大小的内存缓存,会在缓存满时优先删除最早进入缓存的元素，即采用FIFO算法。  

总结:前面说过,LimitedMemoryCache并没有实现createReference(Bitmap)这个方法，所以这个方法的实现都在上面提到的4个类中,它们提供的实现都是WeakReference<Bitmap>,
表面上看似乎能实现弱引用的效果,但实际上根本不会被虚拟机回收,因为LimitedMemoryCache以及它的子类中都还保留了Bitmap的强引用.  

3.1.3 LruMemoryCache  
限制总大小的内存缓存,当缓存满时优先删除最近最少使用的元素。显然它与LRULimitedMemoryCache的作用类似。  
不过要注意的是这里不能采用java库中的LruCache<K,V>来实现,原因在于LruCache<K,V>中的sizeOf(key,value)方法恒返回1而不是Bitmap真正的大小。  
由于LruCache<K,V>中的sizeOf(key,value)方法可以被覆写,故可定义一个继承自LruCache<K,V>的类，重写sizeOf(key,value)方法,然后在LruMemoryCache使用该类的对象,则可使实现更加优雅。  

3.1.4 FuzzyKeyMemoryCache  
根据keyComparator的不同，可以按不同的规则将原本不同的memoryCacheKey看作是相同的。比如默认用的忽略targetSize信息的keyComparator.要注意的是虽然FuzzyKeyMemoryCache做到了put不重复，却没有做到get时返回根据已有的按keyComparator判定相同的key对应的value，在后面会详细讲述。  

3.2 DiskCache家族  
DiskCache家族包含BaseDiskCache,LimitedAgeDiskCache,UnlimitedDiskCache,LruDiskCache.  
3.2.1 BaseDiskCache
实现了DiskCache基本功能的抽象类,实际上它实现了DiskCache的所有功能，但是由于其缺少限制，为了防止它被使用，故将其声明为abstract类。  

1)LimitedAgeDiskCache  
继承自BaseDiskCache,限制缓存文件最长存活时间的DiskCache.在save(...)时记住文件最近修改时间,在get(...)时判断文件存放时间是否超出限制，若超出则删除;  

2)UnlimitedDiskCache  
一个无大小限制的本地图片缓存,继承自BaseDiskCache。与BaseDiskCache完全相同，只是换了个意思明确的类名.  

3)LruDiskCache  
限制总大小的DiskCache,在缓存满时优先删除最近最少使用的文件.它的实现主要依赖JakeWharton的开源项目 [DiskLruCache](https://github.com/JakeWharton/DiskLruCache),后面会有专门的文章介绍。  

3.3 ImageDownloader家族  
ImageDownloader的任务就是要根据不同的Scheme，选择相应的方法进行下载。  
BaseImageDownloader完全实现了ImageDownloader,NetworkDeniedImageDownloader和SlowNetworkImageDownloader都是包装类。   
其中NetworkDeniedImageDownloader是对于网络下载抛出异常，而SlowNetworkDownloader则对于慢网络进行特殊处理，慢网络下的问题见 http://code.google.com/p/android/issues/detail?id=6066

3.4 ImageDecoder  
ImageDecoder的任务就是要根据ImageDecodingInfo进行解码得到Bitmap.  
BaseImageDecoder完全实现了该功能，其解码流程如下。  

![BaseImageDecoder_workflow](http://7xn1yt.com1.z0.glb.clouddn.com/BaseImageDecoder_workflow.png)

显然其最重要的方法就是defineImageSizeAndRotation(...),正是在这个方法中获得了图片的真实大小及翻转信息。  
这里要注意的是图片的ExifInterface包含了很多的信息，比如曝光时间，产商名称，拍摄地点的GPS信息等。感兴趣的同学可以在Google Play下载Exif Viewer或Photo Exif Editor对图片进行分析，也可以到我的github项目上 [ExifSample](https://github.com/HiWong/ExifSample)查看。不过要网络上的大多数图片信息被修改或清除过，比如用大多数手机拍摄的照片是有GPS信息的,但是发到微博或者微信上就没有了,一方面是保护用户隐私，另一方面是减少图片占用空间，对于大的社交平台来说，即使一张图片减少几个字节,总的减少量也是十分可观的。  





