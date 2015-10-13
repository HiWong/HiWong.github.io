---
layout: post
title: "Android中跨进程通信传递Parcelable对象时出现android.os.BadParcelableException: ClassNotFoundException when unmarsh"
date: 2015-10-13 11:06:01 +0800
comments: true
categories: Android
---
按Google开发文档的说法，在跨进程通信时，推荐使用MessengerService而不是AIDL，所以最近在实现一个跨进程的Service时就采用了MessengerService的方法。  

然后定义了这样一个类<!--more-->：  

	public class BleServiceBean implements Parcelable {

	    private String name;

	    private String uuid;

	    public BleServiceBean()
	    {

	    }

	    public BleServiceBean(BluetoothGattService service)
	    {
	        uuid=service.getUuid().toString();
	        name= BleNamesResolver.resolveServiceName(uuid);
	    }

	    private BleServiceBean(Parcel in)
	    {
	        readFromParcel(in);
	    }

	    private void writeToParcel(Parcel out)
	    {
	        out.writeString(name);
	        out.writeString(uuid);
	    }

	    @Override
	    public int describeContents()
	    {
	        return 0;
	    }

	    @Override
	    public void writeToParcel(Parcel out,int flags)
	    {
	        writeToParcel(out);
	    }

	    public void readFromParcel(Parcel in)
	    {
	        name=in.readString();
	        uuid=in.readString();
	    }

	    public static final Parcelable.Creator<BleServiceBean>CREATOR=new
	            Parcelable.Creator<BleServiceBean>(){
	              @Override
	              public BleServiceBean createFromParcel(Parcel source)
	              {
	                  return new BleServiceBean(source);
	              }

	              @Override
	              public BleServiceBean[]newArray(int size)
	              {
	                  return new BleServiceBean[size];
	              }

	            };
	}

在Service中获取到相应数据后要传递给UI，有如下代码：  

	 Bundle bundle=new Bundle();
	 bundle.putParcelableArrayList(BleConnectUtils.BLE_SERVICE_BEAN_LIST,beanList);
	 msg.setData(bundle);
	 mClients.get(i).send(msg);

在Client端获取数据的代码如下：  

	ArrayList<BleServiceBean>beanList=bundle.getParcelableArrayList(BleConnectUtils.BLE_SERVICE_BEAN_LIST);

但是运行到此处时却出现如下错误：  

	FATAL EXCEPTION: main
	    Process: com.example.xxx.xx, PID: 1203
	    android.os.BadParcelableException: ClassNotFoundException when unmarshalling:

给出的出错原因如下：  

	Caused by: java.lang.ClassNotFoundException: Didn't find class "com.example.xxx.xxx.bean.BleServiceBean" on path: DexPathList[[directory "."],nativeLibraryDirectories=[/vendor/lib, /data/cust/lib, /system/lib]]

后面才发现原来是Android有两种不同的classloaders：framework classloader和apk classloader，其中framework classloader知道怎么加载android classes，apk classloader知道怎么加载你的代码，即可以知道你自定义的类，apk classloader继承自framework classloader，所以也知道怎么加载android classes。在应用刚启动时，默认class loader是apk classloader，但在系统内存不足应用被系统回收会再次启动，这个默认class loader会变为framework classloader了，所以对于自己的类会报ClassNotFoundException。  

如果是在要传递的JavaBean中有其中一个Field继承自Parcelable，那么有很简单的处理方法，只要把类似  

	rect = in.readParcelable(null);

改为  

	config = in.readParcelable(Rect.class.getClassLoader());  

但是我们这里是直接传递一个List<BleServiceBean>，那要怎么办呢？  

其实很简单，只需要在Client端读取Bundle中的数据之前加上如下一行代码：  

	bundle.setClassLoader(getClass().getClassLoader());

这样就会使用apk classloader加载。	