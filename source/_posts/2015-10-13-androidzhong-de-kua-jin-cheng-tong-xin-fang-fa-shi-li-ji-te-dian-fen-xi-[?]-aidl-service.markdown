---
layout: post
title: " Android中的跨进程通信方法实例及特点分析(一):AIDL Service"
date: 2015-10-13 15:45:33 +0800
comments: true
categories: Android
---
最近有一个需求就是往程序中加入大数据的采集点，但是因为我们的Android程序包含两个进程，所以涉及到跨进程通信的问题。现将Android中的跨进程通信方式总结如下。  

Android中有4种跨进程通信方式，分别是利用AIDL Service、ContentProvider、Broadcast、Activity实现。  

1.利用AIDL Service实现跨进程通信  

这是我个人比较推崇的方式，因为它相比Broadcast而言，虽然实现上稍微麻烦了一点，但是它的优势就是不会像广播那样在手机中的广播较多时会有明显的时延，甚至有广播发送不成功的情况出现。   

注意普通的Service并不能实现跨进程操作，实际上普通的Service和它所在的应用处于同一个进程<!--more-->中，而且它也不会专门开一条新的线程，因此如果在普通的Service中实现在耗时的任务，需要新开线程。  

要实现跨进程通信，需要借助AIDL(Android Interface Definition Language)。Android中的跨进程服务其实是采用C/S的架构，因而AIDL的目的就是实现通信接口。  

首先举一个简单的栗子。  

服务端代码如下：  

首先是aidl的代码：  

	package com.android.service;

	interface IData
	{
	    int getRoomNum();
	}

RoomService的代码如下：  

	package com.android.service;

	import android.app.Service;
	import android.content.Intent;
	import android.os.IBinder;
	import android.os.RemoteException;

	public class RoomService extends Service{

		private IData.Stub mBinder=new IData.Stub() {
			
			@Override
			public int getRoomNum() throws RemoteException {
				 return 3008;
			}
		};

		@Override
		public IBinder onBind(Intent intent) {
			return mBinder;
		}

	}

AndroidManifest如下：  

	<?xml version="1.0" encoding="utf-8"?>
	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    package="com.android.aidlsampleservice"
	    android:versionCode="1"
	    android:versionName="1.0" >

	    <uses-sdk
	        android:minSdkVersion="8"
	        android:targetSdkVersion="17" />

	    <application
	        android:allowBackup="true"
	        android:icon="@drawable/ic_launcher"
	        android:label="@string/app_name"
	        android:theme="@style/AppTheme" >      
	        <service android:name="com.android.service.RoomService">
	            <intent-filter>
	                <action android:name="com.aidl.service.room"/>
	            </intent-filter>
	        </service>
	    </application>

	</manifest>

然后运行该Service所在的Project即可。  
客户端代码如下：  

注意客户端也要有aidl文件，所以最简单的办法就是将Service端中aidl所在的包直接复制过去。另外要注意的是在onDestroy中要解除和Service的绑定。  

MainActivity.java的代码如下：  

	package com.example.aidlsampleclient;

	import com.android.service.IData;

	import android.os.Bundle;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.app.Activity;
	import android.content.ComponentName;
	import android.content.Intent;
	import android.content.ServiceConnection;
	import android.util.Log;
	import android.view.Menu;
	import android.widget.Button;
	import android.widget.Toast;
	import android.view.View;

	public class MainActivity extends Activity implements View.OnClickListener{

		
		private static final String TAG="MainActivity";
		private static final String ROOM_SERVICE_ACTION="com.aidl.service.room";
		
		private Button bindServiceButton;
		private Button getServiceButton;
		
		
		IData mData;
	    
		private ServiceConnection conn=new ServiceConnection()
		{

			@Override
			public void onServiceConnected(ComponentName name, IBinder service) {
			
				Log.i(TAG,"----------------onServiceConnected--------");
				mData=IData.Stub.asInterface(service);
			}

			@Override
			public void onServiceDisconnected(ComponentName name) {
			
				Log.i(TAG,"----------------onServiceDisconnected-------------");
				mData=null;
				
			}
			
		};
		
		private void initView()
		{
			bindServiceButton=(Button)findViewById(R.id.bindServiceButton);
			getServiceButton=(Button)findViewById(R.id.getServiceButton);
			bindServiceButton.setOnClickListener(this);
			getServiceButton.setOnClickListener(this);
		}
		@Override
		protected void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			initView();
		}
		
		@Override
		public void onClick(View v) {
			// TODO Auto-generated method stub
			switch(v.getId())
			{
			case R.id.bindServiceButton:
				bindService();
				break;
			case R.id.getServiceButton:
				getService();	
			     break;
			default:
				 break;
			}
		}
		
		
		private void bindService()
		{
			Intent intent=new Intent();
			intent.setAction(ROOM_SERVICE_ACTION);
			bindService(intent,conn,BIND_AUTO_CREATE);
		}
		
		private void getService()
		{
			try
			{
				if(mData!=null)
				{
					int roomNum=mData.getRoomNum();
					showLongToast("RoomNum:"+roomNum);
				}
				
			}
			catch(RemoteException ex)
			{
				ex.printStackTrace();
			}
			
		}
		
		private void showLongToast(String info)
		{
			Toast.makeText(getBaseContext(), info, Toast.LENGTH_LONG).show();
		}

		@Override
		public boolean onCreateOptionsMenu(Menu menu) {
			// Inflate the menu; this adds items to the action bar if it is present.
			getMenuInflater().inflate(R.menu.main, menu);
			return true;
		}
		@Override
		protected void onDestroy() {
			// TODO Auto-generated method stub
			super.onDestroy();
			unbindService(conn);
		}
	}

activity_main.xml的代码如下：  

	<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:paddingBottom="@dimen/activity_vertical_margin"
	    android:paddingLeft="@dimen/activity_horizontal_margin"
	    android:paddingRight="@dimen/activity_horizontal_margin"
	    android:paddingTop="@dimen/activity_vertical_margin"
	    tools:context=".MainActivity" >

	    <Button 
	        android:id="@+id/bindServiceButton"
	        android:layout_width="fill_parent"
	        android:layout_height="wrap_content"
	        android:text="BindService"
	        />
	    <Button
	        android:id="@+id/getServiceButton"
	        android:layout_width="fill_parent"
	        android:layout_height="wrap_content"
	        android:text="GetService"
	        android:layout_below="@id/bindServiceButton"
	        />
	       
	</RelativeLayout>

运行结果如下：  

![aidl01](http://7xn1yt.com1.z0.glb.clouddn.com/aidl01.png)

然后举一个稍微复杂一点的栗子，注意如果*.aidl文件中含有自定义的对象，那么该对象的类要实现Parcelable接口，并且要新建一个该类的aidl文件，否则会出现could not find import for class com.android.service.XX的错误，其中XX为类名。还是上面的栗子，但是aidl文件中添加了一些新的方法。仍以上面的RoomService为例，  

Service端的代码如下：  

Room类的代码为：  

	package com.android.service;

	import android.os.Parcel;
	import android.os.Parcelable;

	public class Room implements Parcelable{

		//房间号
		private int roomNum;
		//房间大小
		private float roomSpace;
		//是否有空调
		private boolean hasAirConditioner;
		//是否有Wifi
		private boolean hasWifi;
		//房间内的装饰风格
		private String decorativeStyle;
		
		public static final Parcelable.Creator<Room>CREATOR=new Parcelable.Creator<Room>() {

			@Override
			public Room createFromParcel(Parcel source) {
				return new Room(source);
			}

			@Override
			public Room[] newArray(int size) {
				// TODO Auto-generated method stub
				return null;
			}
			
		};
		
		public Room(int roomNum,float roomSpace,boolean hasAirConditioner,boolean hasWifi,String decorativeStyle)
		{
			this.roomNum=roomNum;
			this.roomSpace=roomSpace;
			this.hasAirConditioner=hasAirConditioner;
			this.hasWifi=hasWifi;
			this.decorativeStyle=decorativeStyle;
		}
		
		private Room(Parcel source)
		{
			roomNum=source.readInt();
			roomSpace=source.readFloat();
			boolean[]tempArray=new boolean[2];
			source.readBooleanArray(tempArray);
		   hasAirConditioner=tempArray[0];
		   hasWifi=tempArray[1];
		   decorativeStyle=source.readString();	
		}
		
		@Override
		public String toString()
		{
			StringBuilder sb=new StringBuilder();
			sb.append("Basic info of room is as follows:\n");
			sb.append("RoomNum:"+roomNum+"\n");
			sb.append("RoomSpace:"+roomSpace+"\n");
			sb.append("HasAirConditioner:"+hasAirConditioner+"\n");
			sb.append("HasWifi:"+hasWifi+"\n");
			sb.append("Decorative Style:"+decorativeStyle);
			return sb.toString();
			
		}
		
		@Override
		public int describeContents() {
			// TODO Auto-generated method stub
			return 0;
		}

		@Override
		public void writeToParcel(Parcel dest,int flags) {
			dest.writeInt(roomNum);
			dest.writeFloat(roomSpace);
			dest.writeBooleanArray(new boolean[]{hasAirConditioner,hasWifi});
			dest.writeString(decorativeStyle);		
		}

	}

Room的声明为：  

	package com.android.service;
	parcelable Room;

IRoom.aidl的代码为:  

	package com.android.service;
	import com.android.service.Room;

	interface IRoom
	{
	  Room getRoom();
	}

RoomService的代码为：  

	package com.android.service;

	import android.app.Service;
	import android.content.Intent;
	import android.os.IBinder;
	import android.os.RemoteException;

	public class RoomService extends Service{

		private IRoom.Stub mBinder=new IRoom.Stub() {
			
			@Override
			public Room getRoom() throws RemoteException {
				Room room=new Room(3008,23.5f,true,true,"IKEA");
				return room;		
			}
		};
		
		@Override
		public IBinder onBind(Intent intent) {
			return mBinder;
		}

	}

由于AndroidManifest.xml的代码不变，因而此处不再贴出。下面是客户端的代码：  

	package com.example.aidlsampleclient;
	import com.android.service.IRoom;
	import android.os.Bundle;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.app.Activity;
	import android.content.ComponentName;
	import android.content.Intent;
	import android.content.ServiceConnection;
	import android.util.Log;
	import android.view.Menu;
	import android.widget.Button;
	import android.widget.Toast;
	import android.view.View;

	public class MainActivity extends Activity implements View.OnClickListener{

		
		private static final String TAG="MainActivity";
		//private static final String SERVICE_ACTION="com.aidl.service.data";
		private static final String ROOM_SERVICE_ACTION="com.aidl.service.room";
		
		private Button bindServiceButton;
		private Button getServiceButton;
		
		
		IRoom mRoom;
	    
		private ServiceConnection conn=new ServiceConnection()
		{

			@Override
			public void onServiceConnected(ComponentName name, IBinder service) {
			
				Log.i(TAG,"----------------onServiceConnected--------");
				showLongToast("onServiceConnected");
				mRoom=IRoom.Stub.asInterface(service);
			}

			@Override
			public void onServiceDisconnected(ComponentName name) {
			
				Log.i(TAG,"----------------onServiceDisconnected-------------");
				mRoom=null;
				
			}
			
		};
		
		private void initView()
		{
			bindServiceButton=(Button)findViewById(R.id.bindServiceButton);
			getServiceButton=(Button)findViewById(R.id.getServiceButton);
			bindServiceButton.setOnClickListener(this);
			getServiceButton.setOnClickListener(this);
		}
		@Override
		protected void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			initView();
		}
		
		@Override
		public void onClick(View v) {
			// TODO Auto-generated method stub
			switch(v.getId())
			{
			case R.id.bindServiceButton:
				bindService();
				break;
			case R.id.getServiceButton:
				getService();	
			     break;
			default:
				 break;
			}
		}
		
		
		private void bindService()
		{
			Intent intent=new Intent();
			intent.setAction(ROOM_SERVICE_ACTION);
			bindService(intent,conn,BIND_AUTO_CREATE);
		}
		
		private void getService()
		{
			if(mRoom!=null)
			{
				try
				{
					showLongToast(mRoom.getRoom().toString());
				}
				catch (RemoteException e) 
				{
					e.printStackTrace();
				}
			}

		}
		
		private void showLongToast(String info)
		{
			Toast.makeText(getBaseContext(), info, Toast.LENGTH_LONG).show();
		}

		@Override
		public boolean onCreateOptionsMenu(Menu menu) {
			// Inflate the menu; this adds items to the action bar if it is present.
			getMenuInflater().inflate(R.menu.main, menu);
			return true;
		}
		@Override
		protected void onDestroy() {
			// TODO Auto-generated method stub
			super.onDestroy();
			unbindService(conn);
		}
	}

注意首先仍然是要将Room,IRoom的代码复制过去，否则会出错。  
运行结果如下：  

![aidl02](http://7xn1yt.com1.z0.glb.clouddn.com/aidl02.png)

显然，客户端已经成功读取到服务信息。  

注意，上面的所举的栗子其实不只是跨进程，还是跨应用。要注意的是，跨应用一定跨进程，但是跨进程不一定是跨应用。对于跨应用的情况，利用AIDL基本上是较好的解决了问题，但也只是“较好”而已，实际上并不完美，比如，如果要增加一个服务，如果利用AIDL的话，那么又要改写aidl文件，如果是涉及自定义对象，则还要增加自定义对象的声明，而且这种改变不只是Service端的改变，客户端也要跟着改变，显然这种解决方案不够优雅。  

那么，有没有更优雅的方法呢？  

当然有，那就是利用Service的onStartCommand(Intent intent, int flags, int startId)方法。  

服务端代码如下：  

	package com.android.service;

	import android.app.Service;
	import android.content.BroadcastReceiver;
	import android.content.Context;
	import android.content.Intent;
	import android.content.IntentFilter;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.util.Log;
	import android.widget.Toast;

	public class RoomService extends Service{

		private static final String TAG="RoomService";
		private static final int CLEAN_SERVICE=0x1;
		private static final int ORDER_SERVICE=0x2;
		private static final int PACKAGE_SERVICE=0x3;
		private static final String SERVICE_KEY="ServiceName";	
		@Override
		public void onStart(Intent intent, int startId) {
		   showLog("onStart");
		}

		@Override
		public int onStartCommand(Intent intent, int flags, int startId) {
			//String action=intent.getAction();
			Log.i(TAG,"onStartCommand");
			
			int actionFlag=intent.getIntExtra(SERVICE_KEY, -1);
			switch(actionFlag)
			{
			case CLEAN_SERVICE:
				showShortToast("Start Clean Service Right Now");
				break;
			case ORDER_SERVICE:
				showShortToast("Start Order Service Right Now");
				break;
			case PACKAGE_SERVICE:
				showShortToast("Start Package Service Right Now");
				break;
			default:
				break;
			}
			return super.onStartCommand(intent, flags, startId);
		}

	    private void showLog(String info)
	    {
	    	Log.i(TAG,info);
	    }
	    
	    private void showShortToast(String info)
	    {
	    	Toast.makeText(getBaseContext(), info, Toast.LENGTH_SHORT).show();
	    }
		@Override
		public void onDestroy() {
			// TODO Auto-generated method stub
			showLog("onDestroy");
			super.onDestroy();
		}

		@Override
		public void onCreate() {
			// TODO Auto-generated method stub
			showLog("onCreate");
			super.onCreate();
		}


		@Override
		public IBinder onBind(Intent intent) {
			showLog("onBind");
			return null;
		}

		@Override
		public boolean onUnbind(Intent intent) {
			showLog("onUnbind");
			return super.onUnbind(intent);
		}
		
	}

客户端代码如下：  

	package com.example.aidlsampleclient;
	import com.android.service.IRoom;
	import com.android.service.RoomService;

	import android.os.Bundle;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.app.Activity;
	import android.content.ComponentName;
	import android.content.Intent;
	import android.content.ServiceConnection;
	import android.util.Log;
	import android.view.Menu;
	import android.widget.Button;
	import android.widget.Toast;
	import android.view.View;

	public class MainActivity extends Activity implements View.OnClickListener{

		private static final String TAG="MainActivity";
		private static final String ROOM_SERVICE_ACTION="com.aidl.service.room";
		
		private static final int CLEAN_SERVICE=0x1;
		private static final int ORDER_SERVICE=0x2;
		private static final int PACKAGE_SERVICE=0x3;
		
		private static final String SERVICE_KEY="ServiceName";
		
		private Button cleanButton;
		private Button orderButton;
		private Button packageButton;
		
		private void initView()
		{
			cleanButton=(Button)findViewById(R.id.cleanButton);
			orderButton=(Button)findViewById(R.id.orderButton);
			packageButton=(Button)findViewById(R.id.packageButton);
		
			cleanButton.setOnClickListener(this);
			orderButton.setOnClickListener(this);
			packageButton.setOnClickListener(this);
		}
		@Override
		protected void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			initView();
		}
		
		@Override
		public void onClick(View v) {
			// TODO Auto-generated method stub
			switch(v.getId())
			{
			case R.id.cleanButton:
			    cleanAction();
			    break;
			case R.id.orderButton:
				orderAction();
				break;
			case R.id.packageButton:
				packageAction();
				break;
			default:
				 break;
			}
		}
			
		private void cleanAction()
		{
			startAction(ROOM_SERVICE_ACTION,CLEAN_SERVICE);
		}
		
		private void orderAction()
		{
			startAction(ROOM_SERVICE_ACTION,ORDER_SERVICE);
		}
		
		private void packageAction()
		{
			startAction(ROOM_SERVICE_ACTION,PACKAGE_SERVICE);
		}
		
		private void startAction(String actionName,int serviceFlag)
		{
			//Intent intent=new Intent(this,RoomService.class);
			Intent intent=new Intent();
			intent.setAction(actionName);
			intent.putExtra(SERVICE_KEY, serviceFlag);
			this.startService(intent);
		}
		
		private void showLongToast(String info)
		{
			Toast.makeText(getBaseContext(), info, Toast.LENGTH_LONG).show();
		}

		@Override
		public boolean onCreateOptionsMenu(Menu menu) {
			// Inflate the menu; this adds items to the action bar if it is present.
			getMenuInflater().inflate(R.menu.main, menu);
			return true;
		}
		@Override
		protected void onDestroy() {
			// TODO Auto-generated method stub
			super.onDestroy();
		}
	}

运行结果如下：  

![aidl03](http://7xn1yt.com1.z0.glb.clouddn.com/aidl03.png)

![aidl04](http://7xn1yt.com1.z0.glb.clouddn.com/aidl04.png)

![aidl05](http://7xn1yt.com1.z0.glb.clouddn.com/aidl05.png)

显然，此时客户端顺利获取了服务。  

上面举的是跨应用的例子，如果是在同一个应用的不同进程的话，则有更简单的实现方法。  

RoomService的代码如下：  

	package com.android.service;

	import com.android.actions.Actions;

	import android.app.Service;
	import android.content.BroadcastReceiver;
	import android.content.Context;
	import android.content.Intent;
	import android.content.IntentFilter;
	import android.os.IBinder;
	import android.os.RemoteException;
	import android.util.Log;
	import android.widget.Toast;

	public class RoomService extends Service{

		private static final String TAG="RoomService";
		@Override
		public void onStart(Intent intent, int startId) {
		   showLog("onStart");
		}

		@Override
		public int onStartCommand(Intent intent, int flags, int startId) {
			//String action=intent.getAction();
			Log.i(TAG,"onStartCommand");
			String action=intent.getAction();
			if(Actions.CLEAN_ACTION.equals(action))
			{    
				showShortToast("Start Clean Service Right Now");
			}
			else if(Actions.ORDER_ACTION.equals(action))
			{
				showShortToast("Start Order Service Right Now");
			}
			else if(Actions.PACKAGE_ACTION.equals(action))
			{
				showShortToast("Start Package Service Right Now");
			}
			else
			{
				showShortToast("Wrong action");
			}
			return super.onStartCommand(intent, flags, startId);
		}

	    private void showLog(String info)
	    {
	    	Log.i(TAG,info);
	    }
	    
	    private void showShortToast(String info)
	    {
	    	Toast.makeText(getBaseContext(), info, Toast.LENGTH_SHORT).show();
	    }
		@Override
		public void onDestroy() {
			// TODO Auto-generated method stub
			showLog("onDestroy");
			super.onDestroy();
		}

		@Override
		public void onCreate() {
			// TODO Auto-generated method stub
			showLog("onCreate");
			super.onCreate();
		}


		@Override
		public IBinder onBind(Intent intent) {
			showLog("onBind");
			return null;
		}

		@Override
		public boolean onUnbind(Intent intent) {
			showLog("onUnbind");
			return super.onUnbind(intent);
		}
		
	}

MainActivity的代码如下：  

	package com.android.activity;
	import com.android.activity.R;
	import com.android.service.RoomService;

	import android.app.Activity;
	import android.content.Intent;
	import android.os.Bundle;
	import android.view.Menu;
	import android.widget.Button;
	import android.widget.Toast;
	import android.view.View;
	import com.android.actions.Actions;

	public class MainActivity extends Activity implements View.OnClickListener{

		private static final String TAG="MainActivity";

		private static final String SERVICE_KEY="ServiceName";
		
		private Button cleanButton;
		private Button orderButton;
		private Button packageButton;
		
		private void initView()
		{
			cleanButton=(Button)findViewById(R.id.cleanButton);
			orderButton=(Button)findViewById(R.id.orderButton);
			packageButton=(Button)findViewById(R.id.packageButton);
		
			cleanButton.setOnClickListener(this);
			orderButton.setOnClickListener(this);
			packageButton.setOnClickListener(this);
		}
		@Override
		protected void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			initView();
		}
		
		@Override
		public void onClick(View v) {
			// TODO Auto-generated method stub
			switch(v.getId())
			{
			case R.id.cleanButton:
			    cleanAction();
			    break;
			case R.id.orderButton:
				orderAction();
				break;
			case R.id.packageButton:
				packageAction();
				break;
			default:
				 break;
			}
		}
			
		private void cleanAction()
		{
			startAction(Actions.CLEAN_ACTION);
		}
		
		private void orderAction()
		{
			startAction(Actions.ORDER_ACTION);
		}
		
		private void packageAction()
		{
			startAction(Actions.PACKAGE_ACTION);
		}
		
		private void startAction(String actionName)
		{
			Intent intent=new Intent(this,RoomService.class);
			intent.setAction(actionName);
			this.startService(intent);
		}
		
		private void showLongToast(String info)
		{
			Toast.makeText(getBaseContext(), info, Toast.LENGTH_LONG).show();
		}

		@Override
		public boolean onCreateOptionsMenu(Menu menu) {
			// Inflate the menu; this adds items to the action bar if it is present.
			getMenuInflater().inflate(R.menu.main, menu);
			return true;
		}
		@Override
		protected void onDestroy() {
			// TODO Auto-generated method stub
			super.onDestroy();
		}
	}

运行结果同上，此处不再贴图。  
那AIDL还有存在的必要吗？当然有!最重要的一点是代价问题，从打印的log可看出，客户端每调用一次Context.startService(Intent)，Service就会重新执行一次onStartCommand---->onStart，而使用AIDL的话，绑定服务之后，不会重复执行onStart，显然后者的代价更小。  

最后再讲一个Service的重点应用：前台Service，像我们经常用的天气、音乐其实都利用了前台Service来实现。  
实例到明天再补上吧。

