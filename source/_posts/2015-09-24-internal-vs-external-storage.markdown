---
layout: post
title: "详解Android中的Internal Storage与External Storage"
date: 2015-09-24 19:37:18 +0800
comments: true
categories: Android
---

**1.Internal storage,External storage与sd card的区别与联系**  

首先要注意的一点就是不要从字面意思上去理解，以为Internal storage即为手机内部存储，External storage是手机外部存储。其实Internal storage和External storage都要从硬件和软件这两个方面去理解。  
 <!--more-->

  **1)硬件方面**   

   Internal storage是Internal memory的一个分区，而External storage较为复杂，它分为两个部分：Primary external storage（可翻译为基础外部存储）和Secondary external storage（可翻译为附加外部存储).其中Primary external storage也是Internal memory的一个分区，而Secondary external storage则是sd card. 显然，这是一个历史遗留问题，在早期Android版本上，由于flash card价格昂贵，所以采用sd card进行拓展，但是后面随着flash card价格下降，并且sd card带来很多问题(比如卡顿，数据容易丢失)，所以Google和手机产商都在逐步取消sd card,到现在大部分手机已经没有sd card了，所以现在的External storage基本可以理解成Internal memory的一个特殊分区。 

  在developer网站上有这么一段话：     
   Sometimes, a device that has allocated a partition of the internal memory for use as the external storage may also offer an SD card slot. When such a device is running Android 4.3 and lower, the getExternalFilesDir() method provides access to only the internal partition and your app cannot read or write to the SD card. Beginning with Android 4.4, however, you can access both locations by calling getExternalFilesDirs(), which returns a File array with entries each location. The first entry in the array is considered the primary external storage and you should use that location unless it's full or unavailable. If you'd like to access both possible locations while also supporting Android 4.3 and lower, use the support library's static method, ContextCompat.getExternalFilesDirs(). This also returns a File array, but always includes only one entry on Android 4.3 and lower.

   Caution Although the directories provided by getExternalFilesDir() and getExternalFilesDirs() are not accessible by the MediaStore content provider, other apps with the READ_EXTERNAL_STORAGE permission can access all files on the external storage, including these. If you need to completely restrict access for your files, you should instead write your files to the internal storage.

----------------------------------------------
  **2)软件方面**
   Internal storage对于app来说是private的，即其他app无权访问，而External    storage对于app来说既可能为public,也可能为private的。   
   具体说就是app的Internal storage位于/data/data/packagename下，而External storage则位于/storagge/emulated/0/下，其中public external storage的位置一般在/storage/emulated/0/dirname下，private external storage则位于/storage/emulated/0/Android/data/下。下面三个图分别为Internal storage,public external storage, private external storage的截图。

   ![Internal storage](http://7xn1yt.com1.z0.glb.clouddn.com/internal_storage.png)
                         图1 Internal storage

   ![public external storage](http://7xn1yt.com1.z0.glb.clouddn.com/external_storage.png)
                         图2 Public external storage
   ![private external storage](http://7xn1yt.com1.z0.glb.clouddn.com/external_private_storage.png)
                         图3 Private external storage

   **3)/storage/emulated/0与/mnt/sdcard的关系**
      /mnt/sdcard其实是/storage/emulated/0的一个软链接(soft link),用过Linux的同学应该都知道，它是为了使用adb命令方便而提供的，如果想利用absolute path在External storage下进行文件操作，应该使用/storage/emulated/0而不是/mnt/sdcard.


**3.Internal storage,External storage的使用**

**1)Internal storage, External storage该如何选择**
 一般来说，Internal storage用来保存不愿被其他app访问而且占用空间较小的信息，卸载时会被删除，比如数据库，设置信息，cache等；
    下图是网易云音乐的Internal storage信息截图：  
  ![internal storage of netease cloud music](http://7xn1yt.com1.z0.glb.clouddn.com/intern_netease.png)


  private external storage用来保存不愿被其他app访问而且占用空间较大的信息，卸载时会被删除，比如下载的图片或者多媒体信息，cache等；
  下图是网易新闻的private external storage信息截图:
 ![private external storage of netease cloud music](http://7xn1yt.com1.z0.glb.clouddn.com/private_extern_netease.png)


 public external storage则用来保存可被其他app访问的信息，如歌曲等；     
 下图是网易云音乐的public external storage的信息截图：   
 ![public external storage of netease cloud music](http://7xn1yt.com1.z0.glb.clouddn.com/public_external_storage_netease.png)

  可以看到歌曲，MV甚至广告cache都在里面。  

**2)Internal storage: getFilesDir(),getCacheDir() 或者直接new File("/data/data/packagename",fileName)**

getFilesDir()
   返回app的internal storage directory，即"/data/data/packagename/files"
getCacheDir()
   返回用于存放临时cache文件的internal storage directory.由于internal storage一般空间比较有限，所以在文件不需要时要及时删除。在空间不足时，系统可能会不经过提醒就删除其中的文件。

new File("data/data/packagename")
    如果想要在/data/data/packagename下创建更多自定义的目录，则需要利用此方法。  

**3)External storage: getExternalFilesDir(),getExternalPublicFilesDir(),getExternalStorageDirectory()**
getExternalFilesDir()
   返回private external storage中相应类型的文件目录，如选择Environment.DIRECTORY_PICTURES类型时，返回/storage/emulated/0/Android/data/packagename/files/Pictures.
   UniversalImageLoader的文件cache放在/storage/emulated/0/Android/data/package/cache下（见StorageUtils中的private static File getExternalCacheDir(Context context))，其实这不是一个好的做法，后面会有文章分析；
getExternalPublicFilesDir()
   返回public external storage相应类型的文件目录,如选择Environment.DIRECTORY_PICTURES类型时，为/storage/emulated/0/Pictures；
getExternalStorageDirectory()
   返回External storage的根目录，即/storage/emulated/0；

**4.示例**

可在我的github下载，地址为：https://github.com/HiWong/StorageSample
下面是在G760上的运行结果：
![output](http://7xn1yt.com1.z0.glb.clouddn.com/output.png)
