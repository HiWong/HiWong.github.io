---
layout: post
title: "高效加载手机中所有图片文件夹的方法"
date: 2016-06-04 21:04:20 +0800
comments: true
categories: android
---
最近在重构代码时有一个业务场景是要加载手机上所有的图片文件夹，并且显示前5张图片进行预览。之前的代码如下<!--more-->：  
``` java private List<ImageBean>getAllImages(Context context)
 /**
   * 得到所有图片
   *
   * @return 返回所有图片列表
   */
  private List<ImageBean> getAllImages(Context context) {
    Log.d(TAG,"start getAllImages");
    List<ImageBean> images_all = new ArrayList<>();
    String where = MediaStore.Images.Media.SIZE + ">=?";
    String columns[] = new String[] {
        MediaStore.Images.Media.DATA, MediaStore.Images.Media._ID, MediaStore.Images.Media.TITLE,
        MediaStore.Images.Media.DISPLAY_NAME, MediaStore.Images.Media.MIME_TYPE
    };
    //use groupby sql to search by folder
    Cursor cursor = context.getContentResolver()
        .query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, columns, where,
            new String[] { 1 * 30 * 1024 + "" }, MediaStore.Images.Media.DATE_ADDED + " desc");
    int photoIndex = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA);
    int nameIndex = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DISPLAY_NAME);
    int idIndex = cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID);

    Log.v("HERE First:", "First Debug");
    ImageBean imageBean;
    while (cursor.moveToNext()) {
      String path = cursor.getString(photoIndex);
      if (!new File(path).exists()) {
        continue;
      }
      long id = cursor.getLong(idIndex);
      imageBean = new ImageBean();
      imageBean.setImage_path(path);
      Cursor thumbCursor = context.getContentResolver()
          .query(MediaStore.Images.Thumbnails.EXTERNAL_CONTENT_URI,
              new String[] { MediaStore.Images.Thumbnails.DATA },
              MediaStore.Images.Thumbnails.IMAGE_ID + "=?", new String[] { String.valueOf(id) },
              null);
      if (thumbCursor.moveToNext()) {
        String thumbnail = thumbCursor.getString(0);
        if (thumbnail != null && !thumbnail.isEmpty()) {
          if (new File(thumbnail).exists()) {
            imageBean.setThumb_path(thumbnail);
          }
        }
      }
      thumbCursor.close();
      images_all.add(imageBean);
    }
    cursor.close();
    Log.d(TAG,"end of getAllImages,size="+images_all.size());
    return images_all;
  }
```
``` java public void photoPartition(List<ImageBean>images_all)
  /**
   * 对图片进行相册分类
   */
  public void photoPartition(List<ImageBean> images_all) {

    for (int i = 0; i < images_all.size(); i++) {
      ImageBean imageBean = images_all.get(i);
      addPhoto(imageBean);
    }
    addedHandler.sendEmptyMessage(0);
  }
```

``` java void addPhoto(ImageBean image)

  /**
   * 添加相册
   */
  public void addPhoto(ImageBean image) {
    File imageFile = new File(image.getImage_path());
    String parentPath = imageFile.getParent();
    //here,we need to use map instead of ArrayList
    int position = photoPathExists(parentPath);
    if (position >= 0) {
      photoList.get(position).getImage_info().add(image);
      photoList.get(position).setPhoto_num(photoList.get(position).getPhoto_num() + 1);
    } else {
      PhotoBean photoBean =
          new PhotoBean(imageFile.getParentFile().getName(), 1, image.getImage_path(), parentPath,
              new ArrayList<ImageBean>() {
              });
      photoBean.getImage_info().add(image);
      photoList.add(photoBean);
    }
  }
```

上述代码的思路非常简单粗暴，就是先遍历MediaStore中所有的图片并记录相关的信息，然后再对遍历到的图片信息按文件夹名称进行分类。对于图片较少的手机来说没什么问题，但是一旦手机上的图片较多的话，就会需要很长的时间。比如在我的手机上有7000多张图片，就需要2分钟以上，这显然不能接受。  

但是对比之下，微信中预览手机的所有图片文件夹则几乎是同步的(实际上并不是同步的，但是由于整个过程耗时很短，只有100ms左右，所以给人的感觉是同步的)。考虑到在MediaStore中查找图片资源其实就是通过sql语句进行查找，所以其实可以通过构建sql语句的查询条件来进行快速搜索。以下代码即可实现加载手机上所有的图片文件夹并且显示封面图片的效果：  
``` java private static List<PhotoBean> getPhotoBeans()
public static List<PhotoBean> getPhotoBeans() {

    List<PhotoBean> photoBeanList = new ArrayList<>();
    String[] mediaColumns = new String[] {
        MediaStore.Images.Media.BUCKET_DISPLAY_NAME, MediaStore.Images.Media.DATA,
        "COUNT(*) AS " + COLUMN_NAME_COUNT
    };
    //SELECT _data, COUNT(*) AS v_count FROM video WHERE ( GROUP BY bucket_display_name)
    //String selection=" 1=1 ) GROUP BY ("+MediaStore.Images.Media.BUCKET_DISPLAY_NAME;
    String selection = MediaStore.Images.Media.SIZE
        + " >=? ) GROUP BY ("
        + MediaStore.Images.Media.BUCKET_DISPLAY_NAME;

    Cursor cursor = TvRemoteApplication.getInstance().getContentResolver()
        .query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, mediaColumns, selection,
            new String[] { String.valueOf(30 * 1024) }, null);
    assert cursor != null;
    while (cursor.moveToNext()) {
      String path = cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.DATA));
      Log.d(TAG, "path:" + path);
      if (path.endsWith(".gif")) {
        continue;
      }

      String bucketName =
          cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.BUCKET_DISPLAY_NAME));
      int count = cursor.getInt(cursor.getColumnIndex(COLUMN_NAME_COUNT));

      photoBeanList.add(new PhotoBean(bucketName, count, path));
    }
    cursor.close();
    //then we need to set first 5 images for every single PhotoBean
    for (PhotoBean bean : photoBeanList) {
      setImageBeansForSinglePhotoBean(bean,5);
    }
    Log.d(TAG, "end of getPhotoBeans,size:" + photoBeanList.size());
    return photoBeanList;
  }
```
由于图片文件夹名称其实就是MediaStore.Images.Media.BUCKET_DISPLAY_NAME,所以可通过 "GROUP BY "+MediaStore.Images.Media.BUCKET_DISPLAY_NAME的方法将图片分成各个子集，然后取出MediaStore.Images.Media.DATA字段即为封面图片。由于数据库的查找十分高效，在我自己的一加手机上即使有7000多张图片，也只需要100ms左右，因而用户甚至感觉不到这个异步的过程。  

由于我们的业务场景跟微信有些不一样，不只是显示封面图片，而是要显示每个图片文件夹的前5张图片(不足5张则显示全部)。并且还要获取每张图片的缩略图，所以还需要对于每个PhotoBean进行一次遍历，根据其bucket_display_name进行条件查询。代码如下：  
```java private static List<ImageBean>getImageBeansForPhotoBean(PhotoBean photoBean,int limitNum)
 private static List<ImageBean> getImageBeansForPhotoBean(PhotoBean photoBean,int limitNum){
    if (null == photoBean) {
      return null;
    }
    ArrayList<ImageBean> imageBeanList = new ArrayList<>();

    String[] mediaColumns = new String[] {
        MediaStore.Images.Media._ID, MediaStore.Images.Media.DATA,
        "(SELECT _data FROM thumbnails WHERE thumbnails.image_id =images._id) AS thumbnail"
    };
    String selection = MediaStore.Images.Media.BUCKET_DISPLAY_NAME
        + " = ? and "
        + MediaStore.Images.Media.SIZE
        + " >=?";

    Cursor cursor = TvRemoteApplication.getInstance()
        .getContentResolver()
        .query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            mediaColumns,
            selection,
            new String[] { photoBean.getFolderName(), String.valueOf(30 * 1024) },
            getSortOrder(limitNum));
    //Cursor cursor = activity.getContentResolver().query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, mediaColumns, selection, new String[]{bucket_name}, MediaStore.Images.Media.DATE_ADDED+" desc");
    assert cursor != null;
    while (cursor.moveToNext()) {
      String data = cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.DATA));
      if (data.endsWith(".gif")) {
        continue;
      }
      String thumbnail = cursor.getString(cursor.getColumnIndex("thumbnail"));
      imageBeanList.add(new ImageBean(data, thumbnail));
    }
    cursor.close();
    return imageBeanList;
  }
  ```
```java private static String getSortOrder(int limitNum)
 private static String getSortOrder(int limitNum){
    if(limitNum>0){
      return MediaStore.Images.Media.DATE_ADDED + " desc limit " + limitNum;
    }
    return MediaStore.Images.Media.DATE_ADDED + " desc";
  }
 ```

最后实现的效果如下:  

![photobean](http://7xn1yt.com1.z0.glb.clouddn.com/photobean.png)

由于相比只显示封面图片的情况，多了一个条件查询的过程，所以在不同手机上的时间表现就不一致了，跟图片文件夹的数量关系比较大，像在我手机上有40多个图片文件夹，最后总耗时约2s左右，虽然没有达到几乎同步的效果，但是相比重构前2min的耗时，已经缩减到了原来的1/60,效率已经提升很多了。  

从这里可以看出微信对于产品的打磨确实是非常用心地，不仅仅是考虑好看，同时也兼顾到了加载效率，这里是一个很典型的设计和技术折衷的选择。  

最后，提醒一下各位做技术的同学，网上有一些所谓“Android高仿微信图片选择器“的文章，其实代码写得特别烂，主要的一个错误就是没有很好地利用GROUP BY去构建sql语句，而是进行了I/O操作，导致耗时很长，跟微信的加载速度完全不能比，希望各位在查找资料时能够擦亮眼睛，细心甄别。  
