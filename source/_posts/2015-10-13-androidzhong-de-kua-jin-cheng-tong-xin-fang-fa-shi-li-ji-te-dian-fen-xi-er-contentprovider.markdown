---
layout: post
title: "Android中的跨进程通信方法实例及特点分析(二):ContentProvider"
date: 2015-10-13 15:27:59 +0800
comments: true
categories: Android
---
1.ContentProvider简介        

在Android中有些数据（如通讯录、音频、视频文件等）是要供很多应用程序使用的，为了更好地对外提供数据，Android系统给我们提供了Content Provider使用，通过它可以访问上面所说的数据，例如很多音乐播放器中的扫描功能其实就用到了Content Provider功能（当然，也有的播放器是自己去实现更底层的功能）。这样的好处是统一管理，比如增加了某个音频文件，底层就会将这种变化通知Content Provider，从而当应用程序访问<!--more-->时就可以获得当前最新的数据。  

当然，Android也允许我们定义自己的Content Provider，只要继承它的基类，并且实现下面的方法即可。  

	public boolean onCreate() 在创建ContentProvider时调用
	public Cursor query(Uri, String[], String, String[], String):用于查询指定Uri的ContentProvider，返回一个Cursor
	public Uri insert(Uri, ContentValues):根据指定的Uri添加数据到ContentProvider中
	public int update(Uri, ContentValues, String, String[]):用于更新指定Uri的ContentProvider中的数据
	public int delete(Uri, String, String[]):根据Uri删除指定的数据
	public String getType(Uri):用于返回指定的Uri中的数据的MIME类型


*如果操作的数据属于集合类型，那么MIME类型字符串应该以vnd.android.cursor.dir/开头。
例如：要得到所有p1记录的Uri为content://contacts/p1，那么返回的MIME类型字符串为"vnd.android.cursor.dir/p1"。
*如果要操作的数据属于非集合类型数据，那么MIME类型字符串应该以vnd.android.cursor.item/开头。
例如：要得到id为100的student记录的Uri为content://contacts/student/100，那么返回的MIME类型字符串应为"vnd.android.cursor.item/student"。  

2.Uri简介  

一个标准的Uri为content://authority/path可分为以下三部分：  

(1)content://:这个部分是ContentProvider规定的，就像http:// 代表Http这个协议一样，使用ContentProvider的协议是content://  

(2)authorities:它在所在的Android系统必须是唯一的，因为系统就是通过它来决定操作或访问哪个ContentProvider的，这与互联网上的网址必须唯一是一样的道理。  

(3)path:资源路径。  

显然，从上面的分析可以看出ContentProvider虽然也可实现跨进程通信，但是它适用的场景主要是与数据库相关，有时也可能是文本文件或XML等存储方式。  

3.ContentResolver  

如果只是定义一个ContentProvider的话，没有任何意义，因为ContentProvider只是内容提供者，它要被别的应用（进程）读取才有价值。与实现ContentProvder的方法相对应，使用ContentResolver相关的方法如下所示：  

	getContentResolver():Context类提供的，用于获取ContentResolver对象；  

	insert(Uri uri,ContentValues values):向Uri对应的ContentProvider中插入values对应的数据；  

	update(Uri uri,ContentValues values,String where,String[]selectionArgs):更新Uri对应的ContentProvider中where处的数据，其中selectionArgs是筛选参数；  

	query(Uri uri,String[]projection,String selection,String[]selectionArgs,String   

	sortOrder):查询Uri对应的ContentProvider中where处的数据，其中selectionArgs是筛选参数，sortOrder是排序方式；  

	delete(Uri uri,String where,String[]selectionArgs):删除Uri对应的ContentProvider中where处的数据，其中selectionArgs是筛选参数；  

4.UriMatcher  

为了确定一个ContentProvider实际能处理的Uri，以及确定每个方法中Uri参数所操作的数据，Android系统提供了UriMatcher工具类。它主要有如下两个方法：  

(1)void addURI(String authority,String path,String code):该方法用于向UriMatcher对象注册Uri。其中authority和path组合成一个Uri，而code则代表该Uri对应的标识码；  

(2)int match(Uri uri):根据前面注册的Uri来判断指定Uri对应的标识码。如果找不到匹配的标识码，该方法将会返回-1。  

下面通过两个实例来讲解ContentProvider的用法，第一个实例是自己定义了一个ContentProvider并且在另一个应用中读取它；第二个实例是读取当前手机中的联系人。  

首先是第一个例子，项目结构如下图所示：  

![content_provider01](http://7xn1yt.com1.z0.glb.clouddn.com/content_provider01.png)

下面是各个类的代码，首先是常量的定义：  

	package com.android.student.utils;

	import android.net.Uri;

	/**
 	- 这里定义了与ContentProvider相关的字符串以及Student对应的数据表中的字段
 	- @author Bettar
	 *
	 */
	public class StudentWords {

		//注意Manifest文件中的authorities属性要跟这里保持一致。
		public static final String AUTHORITY="com.android.student.provider";
		
		public static final Uri STUDENT_WITH_ID_URI=Uri.parse("content://"+AUTHORITY+"/student");
		public static final Uri STUDENT_URI=Uri.parse("content://"+AUTHORITY+"/student");
		
		
		public static final String TABLE_NAME="student";
		public static final String ID="id";
		public static final String NAME="name";
		public static final String SCORE="score";
		public static final String ADDR="address";

	}

然后是数据库帮助类：  

	import com.android.student.utils.StudentWords;

	import android.content.Context;
	import android.database.sqlite.SQLiteDatabase;
	import android.database.sqlite.SQLiteOpenHelper;
	import android.widget.Toast;

	public class StudentDbHelper extends SQLiteOpenHelper{

		private Context context;
		public StudentDbHelper(Context context,String name,int version)
		{
			super(context,name,null,version);
			this.context=context;
		}
		
		@Override
		public void onCreate(SQLiteDatabase db)
		{
			String createSQL="create table "+StudentWords.TABLE_NAME+"("+StudentWords.ID
					+" integer primary key autoincrement,"
					+StudentWords.NAME+" varchar,"
					+StudentWords.SCORE+" integer,"
					+StudentWords.ADDR+" varchar)";
			db.execSQL(createSQL);
		}
		@Override
		public void onUpgrade(SQLiteDatabase db,int oldVersion,int newVersion)
		{
			//升级版本时，可能要执行表结构的修改之类，此处暂时不考虑升级问题，因而只是用Toast提示
			Toast.makeText(context, 
					"newVersion:"+newVersion+" will replace oldVersion:"+oldVersion,
					Toast.LENGTH_LONG).show();	
		}
		
	}

最后是ContentProvider类的定义：  

	package com.android.student.provider;

	import com.android.student.database.StudentDbHelper;

	import android.content.ContentProvider;
	import android.content.ContentUris;
	import android.content.ContentValues;
	import android.content.UriMatcher;
	import android.database.Cursor;
	import android.database.sqlite.SQLiteDatabase;
	import android.net.Uri;
	import android.provider.UserDictionary.Words;

	import com.android.student.utils.StudentWords;

	public class StudentProvider extends ContentProvider{

		private static final String TAG="StudentProvider";
		private static UriMatcher matcher=new UriMatcher(UriMatcher.NO_MATCH);
		private static final int STUDENT_WITH_ID=1;
	    private static final int STUDENT=2;
	    
	    private StudentDbHelper dbHelper;
	    
	    static
	    {
	    	matcher.addURI(StudentWords.AUTHORITY,"student/#",STUDENT_WITH_ID);
	    	//matcher.addURI(StudentWords.AUTHORITY, "student", SINGLE_STUDENT);
	    	//注意其中的#为通配符
	    	matcher.addURI(StudentWords.AUTHORITY, "student", STUDENT);
	    }
	    
	    @Override
	    public boolean onCreate()
	    {
	    	dbHelper=new StudentDbHelper(this.getContext(),"student.db3",1);
	    	return true;
	    }
	    
	    @Override
	    public String getType(Uri uri)
	    {
	    	switch(matcher.match(uri))
	    	{
	    	case STUDENT_WITH_ID:
	    		return "vnd.android.cursor.item/com.android.student";	
	    	case STUDENT:
	    		return "vnd.android.cursor.dir/com.android.student";
	    		default:
	    			throw new IllegalArgumentException("Unknown Uri:"+uri);
	    	}
	    }
	    
	    /**
     	- 由单一的selection这一个筛选条件组合成包含id的复杂筛选条件
     	- @param uri
     	- @param selection
     	- @return
	     */
	    private String getComplexSelection(Uri uri,String selection)
	    {
	    	long id=ContentUris.parseId(uri);
	    	String complexSelection=StudentWords.ID+"="+id;
	    	if(selection!=null&&!"".equals(selection))
	    	{
	    		complexSelection+=" and "+selection;
	    	}
	    	return complexSelection;
	    }
	    
		@Override
		public int delete(Uri uri,String selection,String[]selectionArgs) {
			
			SQLiteDatabase db=dbHelper.getReadableDatabase();
			int num=0;
			switch(matcher.match(uri))
			{
			case STUDENT_WITH_ID:
				String complexSelection=getComplexSelection(uri,selection);
				num=db.delete(StudentWords.TABLE_NAME, complexSelection, selectionArgs);
				break;
			case STUDENT:
				num=db.delete(StudentWords.TABLE_NAME,selection,selectionArgs);
				break;
			default:
				throw new IllegalArgumentException("Unknown Uri:"+uri);
			}
			//通知数据已经改变
			getContext().getContentResolver().notifyChange(uri, null);
			return num;
		}

		@Override
		public Uri insert(Uri uri, ContentValues values) {
			SQLiteDatabase db=dbHelper.getReadableDatabase();
			switch(matcher.match(uri))
			{
			     
				case STUDENT_WITH_ID:
				case STUDENT:
					long rowId=db.insert(StudentWords.TABLE_NAME, StudentWords.ID,values);
					if(rowId>0)
					{
						Uri studentUri=ContentUris.withAppendedId(uri, rowId);
						//如果设置了观察者的话，要通知所有观察者
						getContext().getContentResolver().notifyChange(studentUri, null);
						return studentUri;
					}
					break;
				default:
					throw new IllegalArgumentException("Unknow Uri:"+uri);			
			}
		    return null;
		}

		/**
	 	- 注意其中的projection其实是columns，即列名数组
		 */
		@Override
		public Cursor query(Uri uri, String[] projection, String selection,
				String[] selectionArgs, String sortOrder) {
			SQLiteDatabase db=dbHelper.getReadableDatabase();
			switch(matcher.match(uri))
			{
			//这其实是包含id信息的情况
			case STUDENT_WITH_ID:
				String complexSelection=getComplexSelection(uri,selection);
				return db.query(StudentWords.TABLE_NAME,projection,complexSelection,selectionArgs,null,null,sortOrder);
			//这是不带数字的情况，但是也未必就是好多个，一个也可以。
			case STUDENT:	
				return db.query(StudentWords.TABLE_NAME
						,projection,selection,selectionArgs,null,null,sortOrder);
			default:
					throw new IllegalArgumentException("Unknow Uri"+uri);
			}
		
		}

		@Override
		public int update(Uri uri, ContentValues values, String selection,
				String[] selectionArgs) 
		{
			SQLiteDatabase db=dbHelper.getWritableDatabase();
			int num=0;
			switch(matcher.match(uri))
			{
			case STUDENT_WITH_ID:
				String complexSelection=getComplexSelection(uri,selection);
				num=db.update(StudentWords.TABLE_NAME, values, complexSelection, selectionArgs);	
				break;
			case STUDENT:
				num=db.update(StudentWords.TABLE_NAME, values, selection,selectionArgs);
				break;
				default:
					throw new IllegalArgumentException("Unknow Uri:"+uri);
			}
			
			getContext().getContentResolver().notifyChange(uri,null);
			return num;	
		}
		
		
	}

当然，要记得在Manifest文件中加入ContentProvider的声明：  

 	<provider android:name="com.android.student.provider.StudentProvider"
            android:authorities="com.android.student.provider"
            android:exported="true"/>

下面是对ContentResolver的例子，首先项目中的文件结构如下：  

![content_provider02](http://7xn1yt.com1.z0.glb.clouddn.com/content_provider02.png)

然后是layout文件 ：  

	 <ScrollView 
	      xmlns:android="http://schemas.android.com/apk/res/android"
	      android:layout_width="fill_parent"
	      android:layout_height="wrap_content"     
	      >
	<LinearLayout
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:paddingBottom="@dimen/activity_vertical_margin"
	    android:paddingLeft="@dimen/activity_horizontal_margin"
	    android:paddingRight="@dimen/activity_horizontal_margin"
	    android:paddingTop="@dimen/activity_vertical_margin"
	    android:orientation="vertical"
	    tools:context=".MainActivity" >

	 
	    <TextView 
	        android:id="@+id/tv"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="以下为输入参数:"
	        />
	    <EditText
	        android:id="@+id/nameET"
	        android:layout_width="fill_parent"
	        android:layout_height="wrap_content"
	        android:hint="Name"
	        />
	    <EditText
	        android:id="@+id/scoreET"
	        android:layout_width="fill_parent"
	        android:layout_height="wrap_content"
	        android:hint="Score"
	        />
	    <EditText
	        android:id="@+id/addrET"
	        android:layout_width="fill_parent"
	        android:layout_height="wrap_content"
	        android:hint="Address"
	        />
	    <Button
	        android:id="@+id/insertButton"
	        android:layout_width="fill_parent"
	        android:layout_height="wrap_content"
	        android:text="Insert"
	        />
	      <TextView 
	        android:id="@+id/searchTv"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="以下为查询参数:"
	        />   
	      <EditText 
	          android:id="@+id/inputNameET"
	          android:layout_width="fill_parent"
	          android:layout_height="wrap_content"
	          android:hint="Input name"
	          />
	      <Button 
	          android:id="@+id/searchButton"
	          android:layout_width="fill_parent"
	          android:layout_height="wrap_content"
	          android:text="Search"
	          />
	       <TextView 
	        android:id="@+id/searchResultTv"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="以下为Update参数"
	        />   
	       <EditText 
	          android:id="@+id/inputIdET"
	          android:layout_width="fill_parent"
	          android:layout_height="wrap_content"
	          android:hint="Input id"
	          />
	      <Button 
	          android:id="@+id/updateButton"
	          android:layout_width="fill_parent"
	          android:layout_height="wrap_content"
	          android:text="Update"
	          />
	       <TextView 
	        android:id="@+id/searchResultTv"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="以下为删除参数"
	        />   
	       <EditText 
	          android:id="@+id/inputIdForDeleteET"
	          android:layout_width="fill_parent"
	          android:layout_height="wrap_content"
	          android:hint="Input id"
	          />
	      <Button 
	          android:id="@+id/deleteButton"
	          android:layout_width="fill_parent"
	          android:layout_height="wrap_content"
	          android:text="Delete"
	          />
	 
	</LinearLayout>
	 </ScrollView> 

项目中也有StudentWords这个类，因为与上面的StudentWords完全相同，故此处不再列出,MainActivity的代码如下：  

	package com.android.student.studentcontentresolver;

	import java.util.ArrayList;
	import java.util.List;
	import java.util.Map;

	import com.android.student.utils.StudentWords;

	import android.net.Uri;
	import android.os.Bundle;
	import android.app.Activity;
	import android.content.ContentResolver;
	import android.content.ContentUris;
	import android.content.ContentValues;
	import android.database.Cursor;
	import android.view.Menu;
	import android.widget.Button;
	import android.widget.EditText;
	import android.widget.Toast;
	import android.view.View;
	public class MainActivity extends Activity implements View.OnClickListener{

		ContentResolver contentResolver;
		
		private EditText nameET,scoreET,addressET;
		private Button insertButton;
		private EditText inputNameET;
		private Button searchButton;
		
		private EditText inputIdForUpdateET,inputIdForDeleteET;
		private Button updateButton,deleteButton;
		
		
		private void initView()
		{
			nameET=(EditText)findViewById(R.id.nameET);
			scoreET=(EditText)findViewById(R.id.scoreET);
			addressET=(EditText)findViewById(R.id.addrET);
			insertButton=(Button)findViewById(R.id.insertButton);
			inputNameET=(EditText)findViewById(R.id.inputNameET);
			searchButton=(Button)findViewById(R.id.searchButton);
			
			inputIdForUpdateET=(EditText)findViewById(R.id.inputIdET);
			inputIdForDeleteET=(EditText)findViewById(R.id.inputIdForDeleteET);
			
			updateButton=(Button)findViewById(R.id.updateButton);
			deleteButton=(Button)findViewById(R.id.deleteButton);
			
			insertButton.setOnClickListener(this);
			searchButton.setOnClickListener(this);
			updateButton.setOnClickListener(this);
			deleteButton.setOnClickListener(this);
		}
		
		@Override
		protected void onCreate(Bundle savedInstanceState) {
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			contentResolver=getContentResolver();
			initView();
		}
		
		
		@Override
		public void onClick(View view)
		{
			switch(view.getId())
			{
			case R.id.insertButton:
				insert();
				break;
			case R.id.searchButton:
				query();
				break;
			case R.id.updateButton:
				update();
				break;
			case R.id.deleteButton:
				delete();
				break;
				default:
					break;
			}
		}
		
		
		private void insert()
		{
			String name=nameET.getText().toString();
			String score=scoreET.getText().toString();
			String addr=addressET.getText().toString();
			ContentValues values=new ContentValues();
			values.put(StudentWords.NAME, name);
			values.put(StudentWords.SCORE, new Integer(score));
			values.put(StudentWords.ADDR, addr);
			
			//contentResolver.insert(StudentWords.SINGLE_STUDENT_URI, values);
			//一个是多个的特例，所以此处用MANY_STUDENTS_URI即可。
			contentResolver.insert(StudentWords.STUDENT_URI, values);
	       		
			Toast.makeText(getBaseContext(), "添加学生信息成功", Toast.LENGTH_SHORT).show();		
					
		}
		
		private void query()
		{
			String name=inputNameET.getText().toString();
			//Cursor cursor=contentResolver.query(uri, projection, selection, selectionArgs, sortOrder)
			Cursor cursor=contentResolver.query(StudentWords.STUDENT_URI, null, "name like ? or address like ?", new String[]{"%"+name+"%","%"+name+"%"}, null);
			Toast.makeText(getBaseContext(), getResult(cursor), Toast.LENGTH_LONG).show();
			
		}
		
		private void update()
		{
			//Uri updateUri=StudentWords.SINGLE_STUDENT_URI
			//更新id值为id的记录
			Integer id=new Integer(inputIdForUpdateET.getText().toString());
			Uri updateUri=ContentUris.withAppendedId(StudentWords.STUDENT_WITH_ID_URI,id);
			ContentValues values=new ContentValues();
			values.put(StudentWords.NAME,"VictorWang");
			contentResolver.update(updateUri, values, null, null);
		}
		
		private void delete()
		{
			//删除id值为id的记录
			Integer id=new Integer(inputIdForDeleteET.getText().toString());
			Uri deleteUri=ContentUris.withAppendedId(StudentWords.STUDENT_WITH_ID_URI, id);
			contentResolver.delete(deleteUri, null, null);	
		}
		
		private List<String>convertCursor2List(Cursor cursor)
		{
		    List<String>result=new ArrayList<String>();
			while(cursor.moveToNext())
			{
				result.add(cursor.getString(1)+"  ");
				result.add(cursor.getString(2)+"  ");		
				result.add(cursor.getString(3)+"  ");
			}
			return result;
		}
		
		private String getResult(Cursor cursor)
		{
			StringBuilder sb=new StringBuilder();
			while(cursor.moveToNext())
			{
			     sb.append(cursor.getString(1)+"  ");
			     sb.append(cursor.getString(2)+"  ");
			     sb.append(cursor.getString(3)+"\n");
			}
			return sb.toString();
		}

		@Override
		public boolean onCreateOptionsMenu(Menu menu) {
			// Inflate the menu; this adds items to the action bar if it is present.
			getMenuInflater().inflate(R.menu.main, menu);
			return true;
		}

	}

运行结果如下：  

![content_provider03](http://7xn1yt.com1.z0.glb.clouddn.com/content_provider03.png)

![content_provider04](http://7xn1yt.com1.z0.glb.clouddn.com/content_provider04.png)

今天先写到这里，联系人和ContentObserver的例子后面再添加。