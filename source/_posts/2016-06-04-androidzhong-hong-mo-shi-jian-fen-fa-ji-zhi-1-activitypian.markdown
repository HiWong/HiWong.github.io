---
layout: post
title: "Android中触摸事件分发机制(1):Activity篇"
date: 2016-06-04 21:35:48 +0800
comments: true
categories: android_deep_analysis
---
从这篇文章开始，将会从源码(5.0.1)的角度系统地讲解Android中的事件分发机制。由于这部分内容比较多，所以将会分成三篇，分别是Activity、View和ViewGroup，在最后会给出一份详细的事件分发机制的流程图。<!--more-->  

Activity的事件分发是从dispatchTouchEvent()方法开始的(至于事件分发的完整传递过程，可以看我的博客来了解)，该方法的代码如下：  
``` java dispatchTouchEvent()
public boolean dispatchTouchEvent(MotionEvent ev){
    //可见它是从ACTION_DOWN事件开始分发
	if(ev.getAction()==MotionEvent.ACTION_DOWN){
	    //onUserInteraction()默认为空实现，当然用户可以自己重写这个方法
	    onUserInteraction();
	}
	//这里最终会调用到ViewGroup的dispatchTouchEvent()方法
	if(getWindow().superDispatchTouchEvent(ev)){
	    //可见如果在Activity的根视图中将该事件消费了，则不会调用Activity自身的onTouchEvent()方法，而是直接返回true
	    return true;
	}
	return onTouchEvent(ev);
}
```
在上面的代码中进行了大致的代码注释，下面我们就详细地逐一进行分析。首先是onUserInteraction()方法：  

```java onUserInteraction()
   public void onUserInteraction(){
   }
```
竟然是个空方法，那它的意图就很明显了，就是如果用户想在事件分发之前完成一些操作的话，就可以重写onUserInteraction()方法。但是如果仅仅理解到这个程度的话，就是too young too simple了。实际上onUserInteraction()的作用还不只是这个，它还可以帮助activity很好地管理状态栏，比如帮助activity决定何时取消一个notification.另外，每次回调onUserLeaveHint方法之前，都会回调onUserInteraction，这样保证了activity在用户下拉notification面板并且点击其中的某个item时activity会被告知。  

那什么时候会回调onUserLeaveHint方法呢？如果startActivity时没有添加FLAG_ACTIVITY_NO_USER_ACTION这个flag的话，那么在用户手动离开当前activity时会回调此方法，比如用户主动切换任务、按Home键进入桌面等。而系统自动切换activity时不会调用此方法，比如来电、灭屏等。

另外需要注意的是Activity#onUserInteraction()方法在手势动作的开始，即ACTION_DOWN事件时会被调用，但是在之后的事件(如ACTION_MOVE和ACTION_UP)则不一定会被回调。  

下面是getWindow().superDispatchTouchEvent(ev)，而getWindow()方法如下：  
``` java getWindow()
/**
@return 返回当前Window，如果当前activity还从未显示则返回null
*/
public Window getWindow(){
	return mWindow;
}
```
代码非常简单，就不解释了，那mWindow是何时被创建的呢，看源码发现是在Activity#attach()方法中，下面是attach()方法：  

``` java attach()
	final void attach(Context context, ActivityThread aThread, Instrumentation instr, IBinder token, int ident,
			Application application, Intent intent, ActivityInfo info, CharSequence title, Activity parent, String id,
			NonConfigurationInstances lastNonConfigurationInstances, Configuration config,
			IVoiceInteractor voiceInteractor) {
		attachBaseContext(context);

		mFragments.attachActivity(this, mContainer, null);

		mWindow = PolicyManager.makeNewWindow(this);
		mWindow.setCallback(this);
		mWindow.setOnWindowDismissedCallback(this);
		mWindow.getLayoutInflater().setPrivateFactory(this);
		if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
			mWindow.setSoftInputMode(info.softInputMode);
		}
		if (info.uiOptions != 0) {
			mWindow.setUiOptions(info.uiOptions);
		}
		mUiThread = Thread.currentThread();

		mMainThread = aThread;
		mInstrumentation = instr;
		mToken = token;
		mIdent = ident;
		mApplication = application;
		mIntent = intent;
		mComponent = intent.getComponent();
		mActivityInfo = info;
		mTitle = title;
		mParent = parent;
		mEmbeddedID = id;
		mLastNonConfigurationInstances = lastNonConfigurationInstances;
		if (voiceInteractor != null) {
			if (lastNonConfigurationInstances != null) {
				mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
			} else {
				mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this, Looper.myLooper());
			}
		}

		mWindow.setWindowManager((WindowManager) context.getSystemService(Context.WINDOW_SERVICE), mToken,
				mComponent.flattenToString(), (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
		if (mParent != null) {
			mWindow.setContainer(mParent.getWindow());
		}
		mWindowManager = mWindow.getWindowManager();
		mCurrentConfig = config;
	}

```
注意其中的mWindow=PolicyManager.makeNewWindow(this);语句，PolicyManager这个类非常简单，就全部放在这里了：  

``` java class PolicyManager
public final class PolicyManager {
    private static final String POLICY_IMPL_CLASS_NAME =
        "com.android.internal.policy.impl.Policy";

    private static final IPolicy sPolicy;

    static {
        // Pull in the actual implementation of the policy at run-time
        try {
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);
            sPolicy = (IPolicy)policyClass.newInstance();
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be loaded", ex);
        } catch (InstantiationException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        }
    }

    // Cannot instantiate this class
    private PolicyManager() {}

    // The static methods to spawn new policy-specific objects
    public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }

    public static LayoutInflater makeNewLayoutInflater(Context context) {
        return sPolicy.makeNewLayoutInflater(context);
    }

    public static WindowManagerPolicy makeNewWindowManager() {
        return sPolicy.makeNewWindowManager();
    }

    public static FallbackEventHandler makeNewFallbackEventHandler(Context context) {
        return sPolicy.makeNewFallbackEventHandler(context);
    }
}

```
显然，它是通过反射调用com.android.internal.policy.impl.Policy中的makeNewWindow()方法，该方法如下：  

``` java makeNewWindow()
public Window makeNewWindow(Context context){
	return new PhoneWindow(context);
}
```
到这里就很清楚了:activity中的getWindow()实际上返回的是attach()时创建的PhoneWindow()对象.下面我们就看一下PhoneWindow#superDispatchTouchEvent()方法：  

``` java superDispatchTouchEvent()
@Override
public boolean superDispatchTouchEvent(MotionEvent event){
	return mDecor.superDispatchTouchEvent(event);
}
```
显然，它调用到了mDecor这个对象的superDispatchTouchEvent()方法，那mDecor是什么呢？它其实是一个DecorView对象，它在PhoneWindow中的installDecor()-->generateDecor()方法被创建，代码如下：  

``` java generateDecor()
protected DecorView generateDecor(){
	return new DecorView(getContext(),-1);
}  
```
在了解DecorView之前，我们需要先梳理一下Activity,PhoneWindow,DecorView这三者的关系，为此，我们先看一下Activity的setContentView()方法：  
``` java setContentView(int)
public void setContentView(int layoutResID){
	getWindow().setContentView(layoutResID);
	initWindowDecorActionBar();
}
```
可见首先要调用PhoneWindow#setContentView()方法，该方法主体如下：  
``` java PhoneWindow#setContentView()
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
上面的代码很简单，主要工作就是将layoutResID对应的xml文件通过LayoutInflater对象生成View并且添加至mContentParent视图中。除了大家常用的setContentView(int)外，也可以调用setContentView(View)方法,实际上也是调用PhoneWindow中的setContentView(View)方法，下面我们看一下这个方法：  
``` java setContentView(View)
    @Override
    public void setContentView(View view) {
        setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }

    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            view.setLayoutParams(params);
            final Scene newScene = new Scene(mContentParent, view);
            transitionTo(newScene);
        } else {
            mContentParent.addView(view, params);
        }
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
可见，我们只要看setContentView(View,ViewGroup)这个方法，它跟setContentView(int)差不多，只是不需要inflate过程，最终也是将view添加到mContentParent中。  

关于Activity,PhoneWindow,DecorView的详细分析涉及到的内容太多，后面会单独写一篇博客进行分析。这里就直接给出最终的结论，看下面一张图就一目了然了：  

![activity_phonewindow](http://7xn1yt.com1.z0.glb.clouddn.com/activity_phonewindow.png)

再次回到DecorView#superDispatchTouchEvent(MotionEvent event)方法，如下：  
``` java superDispatchTouchEvent(MotionEvent event)
public boolean superDispatchTouchEvent(MotionEvent event){
	return super.dispatchTouchEvent(event);
}
```
由于DecorView继承自FrameLayout，而FrameLayout继承自ViewGroup,所以super.dispatchTouchEvent(event);实际上是调用ViewGroup#dispatchTouchEvent()方法，再结合前面讲解的Activity,PhoneWindow,DecorView三者的关系可知，事件会从Activity的根视图开始进行分发，最终到我们的自定义的内容视图中。  

下一篇我们就开始分析ViewGroup的事件分发过程.