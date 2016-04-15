---
layout: post
title: "Java服务框架分析"
date: 2016-03-09 01:20:27 +0800
comments: true
categories: android_deep_analysis
---
**引言**
Android服务框架由本地服务框架(Native Service Framework)和Java服务框架(Java Service Framework)两部分组成。前面我们通过Binder机制，详细地分析了本地服务框架的服务注册、获取及使用过程。这篇Blog中我们将分析Java服务框架的架构与实现过程。  

Java服务框架是一系列类的集合，这些类用于支持开发Java系统服务，这些服务运行在基于Java的应用程序框架中。Java服务框架实际上是在本地服务框架的基础上实现的，它通过JNI来使用本地服务框架中的功能，示意图如下<!--more-->：  

![JavaServiceFramework](http://7xn1yt.com1.z0.glb.clouddn.com/Java_Service_Framework.png)

借助JNI，Java层的用户不仅可以使用由Java语言编写的服务，还可以使用由C++语言编写的本地服务。  

1.Java服务框架的层次结构  

跟本地服务框架类似，Java服务框架也是分为三层：服务层、RPC层、IPC层。层次结构示意图如下  

![Java_Native_Service_Layer](http://7xn1yt.com1.z0.glb.clouddn.com/Java_Native_Service_Layer.png)

**服务层**  
与本地服务框架不同，Java服务框架中的服务层需要实现FooManager类，原因是SDK包中并不包含ServiceManager类(实际上源码中是存在这个类的，但是它被标记为@hide，所以不会被编译进SDK中)，应用程序无法使用ServiceManager类来注册或检索系统服务。  

因此，为了让应用程序开发者能够使用系统服务，系统服务开发者就需要将包装类包含到SDK中。例如，为了让应用程序开发者能够使用FooService，系统服务开发者就要编写FooManager包装类，添加使用ServiceManager获取FooService的功能，并将其编入SDK中。之后，应用程序开发者就可以通过包含在SDK中的FooManager类来使用FooService系统服务了。如下图所示：  

![FooManager](http://7xn1yt.com1.z0.glb.clouddn.com/FooManager.png)

**RPC层**  
Java服务框架使用Android平台中的AIDL语言与编译器自动生成服务代理和服务Stub.在Android系统中进程之间不能共享内存，为了使其他应用程序也可以访问本应用程序提供的服务，Android系统采用了远程过程调用的方式来实现，并使用AIDL来公开服务的接口。

**IPC层**  
为了支持Binder IPC，本地服务框架提供了BpBinder与BBinder两个类，Java服务框架则提供了BinderProxy与Binder两个类。  
实际上，Java服务框架是通过JNI使用本地服务框架的Binder IPC，即BinderProxy与Binder通过JNI使用本地服务框架的BpBinder与BBinder类的功能。示意图如下：  

![BinderProxy_IPC](http://7xn1yt.com1.z0.glb.clouddn.com/BinderProxy_IPC.png)

由上图可知，用户调用服务时，首先是BinderProxy的transact()方法使用JNI本地函数调用BpBinder的transact()函数，而后通过Binder IPC调用BBinder的transact()函数。之后BBinder会调用transact()函数，从而引起JavaBBinder对象的回调:onTransact()，而在该方法中会通过JNI调用Binder中的execTransact()方法，在execTransact()中会调用Stub的服务方法，从而最终调用服务。  

2.实现过程  

完整的Java系统服务实现包含注册、获取、调用这3个阶段。下面就按这3个阶段对Java系统服务框架进行分析。为了便于说明，此处以AlarmManagerService为例进行讲解。  

2.1 Java系统服务注册  

在ServiceManager类中，注册服务的代码如下：  

    public static void addService(String name,IBinder service){
        try{
            getIServiceManager().addService(name,service);
        }catch(RemoteException e){
            Log.e(TAG,"error in addService",e);
        }
    }

其中name为服务名称，而service则为服务对象，如AlarmManagerService对象，记住这个很重要，后面会用到的。  

然后看getIServiceManager()方法：  

    private static IServiceManager getIServiceManager(){
        if(sServiceManager!=null){
            return sServiceManager;
        }

        sServiceManager=ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }

首先分析BinderInternal.getContextObject()方法，其声明如下：  

    public static final native IBinder getContextObject();

显然，这是一个native方法，它对应的本地函数是什么呢？在/frameworks/base/core/jni/android_util_Binder.cpp中有如下映射数组：  

    static const JNINativeMethod gBinderInternalMethods[] = {
         /* name, signature, funcPtr */
        { "getContextObject", "()Landroid/os/IBinder;", (void*)android_os_BinderInternal_getContextObject },
        { "joinThreadPool", "()V", (void*)android_os_BinderInternal_joinThreadPool },
        { "disableBackgroundScheduling", "(Z)V", (void*)android_os_BinderInternal_disableBackgroundScheduling },
        { "handleGc", "()V", (void*)android_os_BinderInternal_handleGc }
    };

显然，该native方法对应的本地函数为android_os_BinderInternal_getContextObject()，该函数的定义如下：  

    static jobject android_os_BinderInternal_getContextObject(JNIEnv*env,jobject clazz)
    {
        sp<IBinder>b=ProcessState::self()->getContextObject(NULL);
        return javaObjectForIBinder(env,b);
    }

ProcessState::self()->getContextObject()方法在分析本地服务注册的博客中讲解过，它会返回一个BpBinder对象，而且该BpBinder对象的handle值为0;  
然后进入javaObjectForIBinder()方法：  

    jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
    {
        if (val == NULL) return NULL;

        if (val->checkSubclass(&gBinderOffsets)) {
            // One of our own!
            jobject object = static_cast<JavaBBinder*>(val.get())->object();
            //printf("objectForBinder %p: it's our own %p!\n", val.get(), object);
            return object;
        }

        // For the rest of the function we will hold this lock, to serialize
        // looking/creation of Java proxies for native Binder proxies.
        AutoMutex _l(mProxyLock);

        // Someone else's...  do we know about it?
        jobject object = (jobject)val->findObject(&gBinderProxyOffsets);
        if (object != NULL) {
            jobject res = env->CallObjectMethod(object, gWeakReferenceOffsets.mGet);
            if (res != NULL) {
                LOGV("objectForBinder %p: found existing %p!\n", val.get(), res);
                return res;
            }
            LOGV("Proxy object %p of IBinder %p no longer in working set!!!", object, val.get());
            android_atomic_dec(&gNumProxyRefs);
            val->detachObject(&gBinderProxyOffsets);
            env->DeleteGlobalRef(object);
        }

        object = env->NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
        if (object != NULL) {
            LOGV("objectForBinder %p: created new %p!\n", val.get(), object);
            // The proxy holds a reference to the native object.
            env->SetIntField(object, gBinderProxyOffsets.mObject, (int)val.get());
            val->incStrong(object);

            // The native object needs to hold a weak reference back to the
            // proxy, so we can retrieve the same proxy if it is still active.
            jobject refObject = env->NewGlobalRef(
                    env->GetObjectField(object, gBinderProxyOffsets.mSelf));
            val->attachObject(&gBinderProxyOffsets, refObject,
                    jnienv_to_javavm(env), proxy_cleanup);

            // Note that a new object reference has been created.
            android_atomic_inc(&gNumProxyRefs);
            incRefsCreated(env);
        }

        return object;
    }

首先，前面说过val其实是对应一个handle值为0的BpBinder对象，而BpBinder并没有重写IBinder的checkSubclass()方法，故返回false，从而此处不新建JavaBBinder对象;  
又由于第一次请求时BpBinder的findObject()返回值为NULL,故此处会通过env->NewObject()方法新建BinderProxy对象，并且将BpBinder对象保存到BinderProxy对象中的mObject中，记住这个很重要，后面会再提到。  

再回到BinderInternal中的native IBinder getContextObject()方法中，显然它返回的是一个BinderProxy(BinderProxy继承于IBinder)对象。  

然后回到getIServiceManager()方法中，如下所示：  

    private static IServiceManager getIServiceManager(){
        if(sServiceManager!=null){
            return sServiceManager;
        }

        sServiceManager=ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }

显然，此时ServiceManagerNative.asInterface(BinderInternal.getContextObject());等价于ServiceManagerNative.asInterface(new BinderProxy());下面进入ServiceManagerNative.asInterface()方法：  

    static public IServiceManager asInterface(IBinder obj)
    {
        if(obj==null){
            return null;
        }

        IServiceManager h=(IServiceManager)obj.queryLocalInterface(descriptor);
        if(in!=null){
            return in;
        }

        return new ServiceManagerProxy(obj);

    }

这里的obj其实是BinderProxy对象，而BinderProxy的queryLocalInterface()方法始终返回null,故此处会新建ServiceManagerProxy对象，而ServiceManagerProxy的构造方法特别简单：  

    public ServiceManagerProxy(IBinder remote){
        mRemote=remote;
    }

即将之前新建的BinderProxy对象保存在mRemote中，记住这个很重要，后面还会用到mRemote.  

所以getIServiceManager().addService(name,service);其实调用ServiceManagerProxy中的addService()方法，代码如下：  

    public void addService(String name,IBinder service)
        throws RemoteException{
            Parcel data=Parcel.obtain();
            Parcel reply=Parcel.obtain();
            data.writeInterfaceToken(IServiceManager.descriptor);
            data.writeString(name);
            data.writeStrongBinder(service);
            mRemote.transact(ADD_SERVICE_TRANSACTION,data,reply,0);
            reply.recycle();
            data.recycle();
    }

+ 该方法中首先是将RPC数据写入到data中，接口描述符和服务名称都很好理解，关键是data.writeStringBinder(service);注意service是AlarmManagerService对象。  
Parcel类的writeStrongBinder()方法如下：  

    public final native void writeStrongBinder(IBinder val);

显然，它是一个native方法，类似地，在/frameworks/base/core/jni/android_util_Binder.cpp中可以找到它对应的本地函数是android_os_Parcel_writeStrongBinder(),其代码如下(注意传入的参数clazz是Java中的Parcel类对象，object则是AlarmManagerService对象)：  

    static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jobject clazz, jobject object)
    {
        Parcel* parcel = parcelForJavaObject(env, clazz);
        if (parcel != NULL) {
            const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
            if (err != NO_ERROR) {
                jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
            }
        }
    }

其中parcelForJavaObject()的目的是将Parcel(Java类)中的mObject转化为Parcel(C++)指针。  
在分析Parcel(C++)类的writeStrongBinder()方法之前，我们先分析ibinderForJavaObject()方法，注意此处传入的参数obj是AlarmManagerService对象：  

    sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
    {
        if (obj == NULL) return NULL;

        if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
            JavaBBinderHolder* jbh = (JavaBBinderHolder*)
                env->GetIntField(obj, gBinderOffsets.mObject);
            return jbh != NULL ? jbh->get(env) : NULL;
        }

        if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
            return (IBinder*)
                env->GetIntField(obj, gBinderProxyOffsets.mObject);
        }

        LOGW("ibinderForJavaObject: %p is not a Binder object", obj);
        return NULL;
    }

由于AlarmManagerService继承于IAlarmManager.Stub，而IAlarmManager.Stub继承于Binder，而gBinderOffsets.mClass对应的正是Binder类，故env->IsInstanceOf(obj,gBinderOffsets.mClass)这个判断成立，从而会新建JavaBBinderHolder类，其构造函数如下：  

    JavaBBinderHolder(JNIEnv*env,jobject object)
        :mObject(object)
    {
        LOGV("Creating JavaBBinderHolder for Object %p\n",object);
    }

显然，AlarmManagerService对象被赋值给了JavaBBinderHolder中的mObject成员;  

之后返回jbh->get(evn)，而JavaBBinder的get()方法如下：  

    sp<JavaBBinder> get(JNIEnv* env)
    {
        AutoMutex _l(mLock);
        sp<JavaBBinder> b = mBinder.promote();
        if (b == NULL) {
            b = new JavaBBinder(env, mObject);
            mBinder = b;
            LOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%d\n",
                 b.get(), b->getWeakRefs(), mObject, b->getWeakRefs()->getWeakCount());
        }

        return b;
    }

显然这里采用了懒加载的方式，在第一次使用时新建了JavaBBinder对象，注意前面说到的mObject，它实际上对应AlarmManagerService对象。  
而JavaBBinder的构造函数如下：  

    JavaBBinder(JNIEnv* env, jobject object)
        : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
    {
        LOGV("Creating JavaBBinder %p\n", this);
        android_atomic_inc(&gNumLocalRefs);
        incRefsCreated(env);
    }

首先jnienv_to_javavm()函数的作用是通过jni环境获得java虚拟机并且赋值给mVm;  
之后为object(AlarmManagerService对象的本地引用)创建全局引用，这种对象如果不主动释放，它永远都不会被垃圾回收，这样做的目的是为了让AlarmManagerService在手机开机时一直保持运行。  

再回到ibinderForJavaObject()这个方法中，通过上面的分析我们知道它返回一个JavaBBinder对象，并且该对象中的mObject保存着AlarmManagerService对象的本地引用;  

再回到android_os_Parcel_writeStrongBinder()方法中，parcel->writeStrongBinder(ibinderForJavaObject(env, object))就等价于parcel->writeStrongBinder(new JavaBBinder());**注意这里是和单纯的本地系统服务注册过程中不同的地方，单纯的本地系统服务中在writeStrongBinder(service);中的service是本地服务对象，如MediaPlayerService对象，而这里用的是JavaBBinder，它有点像一个包装类，将Java服务对象(如AlarmManagerService对象)包装在它的成员mObject中。这个mObject后面还会提到.**  

而writeStrongBinder()方法的代码如下：  

    status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
    {
        return flatten_binder(Process::self(),val,this);
    }

注意这里传入的参数val是JavaBBinder对象。flatten_binder()方法如下：  

    status_t flatten_binder(const sp<ProcessState>& proc,
        const sp<IBinder>& binder, Parcel* out)
    {
        flat_binder_object obj;
        
        obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
        if (binder != NULL) {
            IBinder *local = binder->localBinder();
            if (!local) {
                BpBinder *proxy = binder->remoteBinder();
                if (proxy == NULL) {
                    LOGE("null proxy");
                }
                const int32_t handle = proxy ? proxy->handle() : 0;
                obj.type = BINDER_TYPE_HANDLE;
                obj.handle = handle;
                obj.cookie = NULL;
            } else {
                obj.type = BINDER_TYPE_BINDER;
                obj.binder = local->getWeakRefs();
                obj.cookie = local;
            }
        } else {
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = NULL;
            obj.cookie = NULL;
        }
        
        return finish_flatten_binder(binder, obj, out);
    }

这里的binder其实是JavaBBinder对象，而JavaBBinder继承于BBinder,它重写了localBinder()方法，返回this,所以这里执行的是如下代码：  

     obj.type = BINDER_TYPE_BINDER;
     obj.binder = local->getWeakRefs();
     obj.cookie = local;

所以flat_binder_object对象obj的成员type为BINDER_TYPE_BINDER,成员binder则为JavaBBinder对象的弱引用，成员cookie则保存着JavaBBinder对象，特别注意cookie成员，后面还会提到。  
之后Parcel方法的调用与单纯的本地系统服务一致，不再赘述。  

===============================================================================================================================
再回到addService()方法中：  

     public void addService(String name,IBinder service)
        throws RemoteException{
            Parcel data=Parcel.obtain();
            Parcel reply=Parcel.obtain();
            data.writeInterfaceToken(IServiceManager.descriptor);
            data.writeString(name);
            data.writeStrongBinder(service);
            mRemote.transact(ADD_SERVICE_TRANSACTION,data,reply,0);
            reply.recycle();
            data.recycle();
    }

mRemote在之前已经说过，它是一个BinderProxy对象，所以mRemote.transact()实际上调用的是BinderProxy的transact()方法，代码如下：  

    public native boolean transact(int code,Parcel data,Parcel reply,int flags) throws RemoteException;

显然，这是一个native方法，它对应的本地函数也可以在/frameworks/base/core/jni/android_util_Binder.cpp中找到，对应的函数是android_os_BinderProxy_transact()，如下所示，注意传入的参数obj为BinderProxy对象，code为ADD_SERVICE_TRANSACTION,dataObj为上面的Parcel对象data的本地引用(其中data中含有IServiceManager接口描述符,服务名称以及AlarmManagerService对象),replyObj则为Parcel对象的本地引用：  

    static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
                                                jint code, jobject dataObj,
                                                jobject replyObj, jint flags)
    {
        if (dataObj == NULL) {
            jniThrowException(env, "java/lang/NullPointerException", NULL);
            return JNI_FALSE;
        }

        Parcel* data = parcelForJavaObject(env, dataObj);
        if (data == NULL) {
            return JNI_FALSE;
        }
        Parcel* reply = parcelForJavaObject(env, replyObj);
        if (reply == NULL && replyObj != NULL) {
            return JNI_FALSE;
        }

        IBinder* target = (IBinder*)
            env->GetIntField(obj, gBinderProxyOffsets.mObject);
        if (target == NULL) {
            jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
            return JNI_FALSE;
        }

        LOGV("Java code calling transact on %p in Java object %p with code %d\n",
                target, obj, code);

        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        const bool time_binder_calls = should_time_binder_calls();

        int64_t start_millis;
        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
        //printf("Transact from Java code to %p sending: ", target); data->print();
        status_t err = target->transact(code, *data, reply, flags);
        //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }

        if (err == NO_ERROR) {
            return JNI_TRUE;
        } else if (err == UNKNOWN_TRANSACTION) {
            return JNI_FALSE;
        }

        signalExceptionForError(env, obj, err);
        return JNI_FALSE;
    }

前面的代码都是将Java中的Parcel对象转化为C++中的Parcel对象，之后获取BinderProxy中的mObject成员，前面说过，mObject成员保存的是BpBinder对象，所以这里IBinder*target其实指向的是BpBinder对象。从而target->transact()其实调用的是BpBinder的transact()方法，该方法的代码如下：  

    status_t BpBinder::transact(
        uint32_t code,const Parcel& data,Parcel* reply,uint32_t flags)
    {
        if(mAlive){
            status_t status=IPCThreadState::self()->transact(mHandle,code,data,reply,flags);
            if(status==DEAD_OBJECT) mAlive=0;
            return status;
        }
        return DEAD_OBJECT;
    }

然后进入IPCThreadState的transact()方法，其主要代码如下：  

    status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
    {
        status_t err = data.errorCheck();

        flags |= TF_ACCEPT_FDS;
       
        if (err == NO_ERROR) {
            
            err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
        }
        
        if (err != NO_ERROR) {
            if (reply) reply->setError(err);
            return (mLastError = err);
        }
        
        if ((flags & TF_ONE_WAY) == 0) {
            if (reply) {
                //we will get reply from this function.
                err = waitForResponse(reply);
            } else {
                Parcel fakeReply;
                err = waitForResponse(&fakeReply);
            }
            
        } else {
            err = waitForResponse(NULL, NULL);
        }
        
        return err;
    }

然后进入writeTransactionData()函数中，注意传入的参数cmd为BC_TRANSACTION,binderFlags为flags(0),handle为0,code为ADD_SERVICE_TRANSACTION,data为上面的Parcel对象,statusBuffer为NULL：  

    status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
    {
        binder_transaction_data tr;

        tr.target.handle = handle;
        tr.code = code;
        tr.flags = binderFlags;
        
        const status_t err = data.errorCheck();
        if (err == NO_ERROR) {
            tr.data_size = data.ipcDataSize();
            tr.data.ptr.buffer = data.ipcData();
            tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
            tr.data.ptr.offsets = data.ipcObjects();
        } else if (statusBuffer) {
           ...
        } else {
            return (mLastError = err);
        }
        
        mOut.writeInt32(cmd);
        mOut.write(&tr, sizeof(tr));
        
        return NO_ERROR;
    }

显然，这里主要就是将IPC数据写入binder_transaction_data对象中,并且将cmd(此处值为BC_TRANSACTION)写入到mOut中.  

再回到IPCThreadState::transact()方法中，之后执行waitForResponse()方法中,其代码如下：  

    status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
    {
        int32_t cmd;
        int32_t err;

        while (1) {
            if ((err=talkWithDriver()) < NO_ERROR) break;
            err = mIn.errorCheck();
            if (err < NO_ERROR) break;
            if (mIn.dataAvail() == 0) continue;
            
            cmd = mIn.readInt32();

            switch (cmd) {
            case BR_TRANSACTION_COMPLETE:
                if (!reply && !acquireResult) goto finish;
                break;
            
            case BR_DEAD_REPLY:
                err = DEAD_OBJECT;
                goto finish;

            case BR_FAILED_REPLY:
                err = FAILED_TRANSACTION;
                goto finish;
            
            case BR_ACQUIRE_RESULT:
                {
                    LOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                    const int32_t result = mIn.readInt32();
                    if (!acquireResult) continue;
                    *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
                }
                goto finish;
            
            case BR_REPLY:
                {
                    binder_transaction_data tr;
                    err = mIn.read(&tr, sizeof(tr));
                    LOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                    if (err != NO_ERROR) goto finish;

                    if (reply) {
                        if ((tr.flags & TF_STATUS_CODE) == 0) {
                            reply->ipcSetDataReference(
                                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                                tr.data_size,
                                reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                                tr.offsets_size/sizeof(size_t),
                                freeBuffer, this);
                        } else {
                            err = *static_cast<const status_t*>(tr.data.ptr.buffer);
                            freeBuffer(NULL,
                                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                                tr.data_size,
                                reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                                tr.offsets_size/sizeof(size_t), this);
                        }
                    } else {
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(size_t), this);
                        continue;
                    }
                }
                goto finish;

            default:
                err = executeCommand(cmd);
                if (err != NO_ERROR) goto finish;
                break;
            }
        }

    finish:
        if (err != NO_ERROR) {
            if (acquireResult) *acquireResult = err;
            if (reply) reply->setError(err);
            mLastError = err;
        }
        
        return err;
    }

在博客[本地服务的注册过程](http://blog.imallen.wang/blog/2016/02/27/android-binderji-zhi-3-ben-di-fu-wu-zhu-ce-guo-cheng/)这篇博客中，我们提到了talkWithDriver()的作用，就是将保存在mOut中的Binder IPC数据传递给Binder Driver，并将来自Binder Driver的Binder IPC数据保存在mIn中，同时会新建binder_node.  

在talkWithDriver()中新建完binder_node之后，Binder IPC协议就变成BR_REPLY,所以此时tr中包含的数据就是BR_REPLY和binder_transaction_data对象，之后通过调用ipcSetDataReference()方法将注册AlarmManagerService的结果存入到Parcel对象reply中，至此，添加服务的过程就完成了。  

2.2 Java系统服务获取  

当Context对象(如Activity)调用getSystemService(Context.ALARM_SERVICE)方法时，实际调用的是ContextImpl中的getSystemService()方法，该方法如下：  

        @Override
    public Object getSystemService(String name) {
        if (WINDOW_SERVICE.equals(name)) {
            return WindowManagerImpl.getDefault();
        } else if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            synchronized (mSync) {
                LayoutInflater inflater = mLayoutInflater;
                if (inflater != null) {
                    return inflater;
                }
                mLayoutInflater = inflater =
                    PolicyManager.makeNewLayoutInflater(getOuterContext());
                return inflater;
            }
        } else if (ACTIVITY_SERVICE.equals(name)) {
            return getActivityManager();
        } else if (INPUT_METHOD_SERVICE.equals(name)) {
            return InputMethodManager.getInstance(this);
        } else if (ALARM_SERVICE.equals(name)) {
            return getAlarmManager();
        } else if (ACCOUNT_SERVICE.equals(name)) {
            return getAccountManager();
        } else if (POWER_SERVICE.equals(name)) {
            return getPowerManager();
        } else if (CONNECTIVITY_SERVICE.equals(name)) {
            return getConnectivityManager();
        } else if (THROTTLE_SERVICE.equals(name)) {
            return getThrottleManager();
        } else if (WIFI_SERVICE.equals(name)) {
            return getWifiManager();
        } else if (NOTIFICATION_SERVICE.equals(name)) {
            return getNotificationManager();
        } else if (KEYGUARD_SERVICE.equals(name)) {
            return new KeyguardManager();
        } else if (ACCESSIBILITY_SERVICE.equals(name)) {
            return AccessibilityManager.getInstance(this);
        } else if (LOCATION_SERVICE.equals(name)) {
            return getLocationManager();
        } else if (SEARCH_SERVICE.equals(name)) {
            return getSearchManager();
        } else if (SENSOR_SERVICE.equals(name)) {
            return getSensorManager();
        } else if (STORAGE_SERVICE.equals(name)) {
            return getStorageManager();
        } else if (VIBRATOR_SERVICE.equals(name)) {
            return getVibrator();
        } else if (STATUS_BAR_SERVICE.equals(name)) {
            synchronized (mSync) {
                if (mStatusBarManager == null) {
                    mStatusBarManager = new StatusBarManager(getOuterContext());
                }
                return mStatusBarManager;
            }
        } else if (AUDIO_SERVICE.equals(name)) {
            return getAudioManager();
        } else if (TELEPHONY_SERVICE.equals(name)) {
            return getTelephonyManager();
        } else if (CLIPBOARD_SERVICE.equals(name)) {
            return getClipboardManager();
        } else if (WALLPAPER_SERVICE.equals(name)) {
            return getWallpaperManager();
        } else if (DROPBOX_SERVICE.equals(name)) {
            return getDropBoxManager();
        } else if (DEVICE_POLICY_SERVICE.equals(name)) {
            return getDevicePolicyManager();
        } else if (UI_MODE_SERVICE.equals(name)) {
            return getUiModeManager();
        }

        return null;
    }

由于传入的参数为Context.ALARM_SERVICE,进入getAlarmManager()方法中：  

    private AlarmManager getAlarmManager() {
        synchronized (sSync) {
            if (sAlarmManager == null) {
                IBinder b = ServiceManager.getService(ALARM_SERVICE);
                IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                sAlarmManager = new AlarmManager(service);
            }
        }
        return sAlarmManager;
    }

首先分析ServiceManager.getService()这个方法：  

    public static IBinder getService(String name)
    {
        try
        {
            IBinder service=cCache.get(name);
            if(service!=null){
                return service;
            }else if{
                return getIServiceManager().getService(name);
            }
        }catch(RemoteException e){
            Log.e(TAG,"error in getService",e);
        }
        return null;
    }

如果是第一次请求该服务，则没有缓存，从而调用getIServiceManager().getService()方法。  

在上面的服务注册过程中分析了getIServiceManager()其实返回的是ServiceManagerProxy对象，并且该ServiceManagerProxy对象中的mRemote保存着BinderProxy对象。  

再回到ServiceManager.getService()方法中，可知getIServiceManager().getService()其实调用的是ServiceManagerProxy的getService()方法，该方法的代码如下：  

    public IBinder getService(String name) throws RemoteException{
        Parcel data=Parcel.obtain();
        Parcel reply=Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        mRemote.transact(GET_SERVICE_TRANSACTION,data,reply,0);
        IBinder binder=reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }

这个方法里，重点是mRemote.transact(GET_SERVICE_TRANSACTION,data,reply,0);这个语句，前面说过，mRemote是一个BinderProxy对象，所以这里其实调用的是BinderProxy的transact()方法，代码如下：  

    public native boolean transact(int code,Parcel data,Parcel reply,int flags) throws RemoteException;

显然，这是一个native方法，它对应的本地函数也可以在/frameworks/base/core/jni/android_util_Binder.cpp中找到，对应的函数是android_os_BinderProxy_transact()，如下所示：  

    static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
                                                jint code, jobject dataObj,
                                                jobject replyObj, jint flags)
    {
        if (dataObj == NULL) {
            jniThrowException(env, "java/lang/NullPointerException", NULL);
            return JNI_FALSE;
        }

        Parcel* data = parcelForJavaObject(env, dataObj);
        if (data == NULL) {
            return JNI_FALSE;
        }
        Parcel* reply = parcelForJavaObject(env, replyObj);
        if (reply == NULL && replyObj != NULL) {
            return JNI_FALSE;
        }

        IBinder* target = (IBinder*)
            env->GetIntField(obj, gBinderProxyOffsets.mObject);
        if (target == NULL) {
            jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
            return JNI_FALSE;
        }

        LOGV("Java code calling transact on %p in Java object %p with code %d\n",
                target, obj, code);

        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        const bool time_binder_calls = should_time_binder_calls();

        int64_t start_millis;
        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
        //printf("Transact from Java code to %p sending: ", target); data->print();
        status_t err = target->transact(code, *data, reply, flags);
        //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }

        if (err == NO_ERROR) {
            return JNI_TRUE;
        } else if (err == UNKNOWN_TRANSACTION) {
            return JNI_FALSE;
        }

        signalExceptionForError(env, obj, err);
        return JNI_FALSE;
    }

前面的代码都是将Java中的Parcel对象转化为C++中的Parcel对象，之后获取BinderProxy中的mObject成员，前面说过，mObject成员保存的是BpBinder对象，所以这里IBinder*target其实指向的是BpBinder对象。从而target->transact()其实调用的是BpBinder的transact()方法，之后就和本地服务框架中的获取服务过程一样了，最后是将返回的数据放在reply中。  

返回到getService()方法中，下面要调用的是reply.readStrongBinder();该方法的定义如下：  

    public final native IBinder readStrongBinder();

显然，该方法是一个native方法，它对应的是android_os_Parcel_readStrongBinder()函数：  

    static jobject android_os_Parcel_readStrongBinder(JNIEnv*env,jobject clazz)
    {
        Parcel*parcel=parcelForJavaObject(env,clazz);
        if(parcel!=null){
            return javaObjectForIBinder(env,parcel->readStrongBinder());
        }
        return NULL;
    }

在获取本地服务的博客中我们分析过，readStrongBinder()实际上返回的是一个持有相应服务handle值的BpBinder对象(注意，这个BpBinder对象和前面的BpBinder对象不同，前面的BpBinder持有的handle值为0，而此处为AlarmManagerService注册的handle值).  

而javaObjectForIBinder()函数在前面已经分析过，它会生成一个BinderProxy对象，并且将刚刚生成的BpBinder对象保存在BinderProxy中的mObject中.

这样，getService()方法就分析完了，最终结果是返回BinderProxy对象，并且该对象中的mObject保存着对应于AlarmManagerService的BpBinder对象.  

下面再回到getAlarmManager()中：  

    private AlarmManager getAlarmManager() {
        synchronized (sSync) {
            if (sAlarmManager == null) {
                IBinder b = ServiceManager.getService(ALARM_SERVICE);
                IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                sAlarmManager = new AlarmManager(service);
            }
        }
        return sAlarmManager;
    }

下面分析IAlarmManager.Stub.asInterface()方法：  

    public static android.app.IAlarmManager asInterface(android.os.IBinder obj)
    {
        if(obj==null){
            return null;
        }
        android.os.IInterface iin=(android.os.IInterface)obj.queryLocalInterface(DESCRIPTOR);
        if((iin!=null)&&(iin instanceof android.app.IAlarmManager)){
            return ((android.app.IAlarmManager)iin);
        }
        return new android.app.IAlarmManager.Stub.Proxy(obj);
    }

前面说过，这个obj其实是BinderProxy对象，前面说过，BinderProxy的queryLocalInterface()始终返回null,所以这里会新建IAlarmManager.Stub.Proxy对象，其构造方法如下：  

    Proxy(android.os.IBinder remote)
    {
        mRemote=remote;
    }

即将BinderProxy对象保存在Proxy中的mRemote成员中，记住这个，后面会用到。  

再回到getAlarmManager()方法中，可见service其实是一个IAlarmManager.Stub.Proxy对象。下面一句很简单，就是新建AlarmManager对象，其构造方法如下：  

    AlarmManager(IAlarmManager service){
        mService=service;
    }

可见，将前面新建的IAlarmManager.Stub.Proxy对象保存在AlarmManager中的mService对象中。  

到这里，我们就成功获取到了AlarmManager对象，并且AlarmManager对象中的mService保存着IAlarmManager.Stub.Proxy对象，而Proxy对象中保存着BinderProxy对象，BinderProxy对象中保存着持有AlarmManagerService的handle值的BpBinder(C++)对象.  

由于BpBinder持有了AlarmManagerService的handle值，所以到这里就相当于获取到了我们需要的闹钟服务。  

2.3 Java系统服务调用  

获取到服务后，返回AlarmManager对象，调用闹钟服务的过程示意图如下所示：  

![Alarm_Manager_Service](http://7xn1yt.com1.z0.glb.clouddn.com/Alarm_Manager_Service.png)

获取了AlarmManager对象之后，比如调用setTime()方法，代码如下：  

    public void setTime(long millis){
        try{
            mService.setTime(millis);
        }catch(RemoteException ex){
        }
    }

前面说过，mService其实是IAlarmManager.Stub.Proxy对象，所以mService.setTime(millis);其实是调用IAlarmManager.Stub.Proxy的setTime()方法，代码如下所示：  

    public void setTime(long millis) throws android.os.RemoteException
    {
        android.os.Parcel _data=android.os.Parcel.obtain();
        android.os.Parcel _reply=android.os.Parcel.obtain();

        try{
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeLong(millis);
            mRemote.transact(Stub.TRANSACTION_setTime,_data,_reply,0);
            _reply.readException();
        }finally{
            _reply.recycle();
            _data.recycle();
        }
    }

前面说过，保存在Proxy对象中的mRemote中保存的是BinderProxy对象，而BinderProxy的transact()是个native方法，它对应的本地函数为android_os_BinderProxy_transact()，注意传入的参数obj为BinderProxy对象的本地引用,code为Stub.TRANSACTION_setTime,dataObj为保存Binder IPC数据的Parcel对象,replyObj则用于存放返回结果:  

     static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
                                                jint code, jobject dataObj,
                                                jobject replyObj, jint flags)
    {
        if (dataObj == NULL) {
            jniThrowException(env, "java/lang/NullPointerException", NULL);
            return JNI_FALSE;
        }

        Parcel* data = parcelForJavaObject(env, dataObj);
        if (data == NULL) {
            return JNI_FALSE;
        }
        Parcel* reply = parcelForJavaObject(env, replyObj);
        if (reply == NULL && replyObj != NULL) {
            return JNI_FALSE;
        }

        IBinder* target = (IBinder*)
            env->GetIntField(obj, gBinderProxyOffsets.mObject);
        if (target == NULL) {
            jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
            return JNI_FALSE;
        }

        LOGV("Java code calling transact on %p in Java object %p with code %d\n",
                target, obj, code);

        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        const bool time_binder_calls = should_time_binder_calls();

        int64_t start_millis;
        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
        //printf("Transact from Java code to %p sending: ", target); data->print();
        status_t err = target->transact(code, *data, reply, flags);
        //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }

        if (err == NO_ERROR) {
            return JNI_TRUE;
        } else if (err == UNKNOWN_TRANSACTION) {
            return JNI_FALSE;
        }

        signalExceptionForError(env, obj, err);
        return JNI_FALSE;
    }

显然，这里会调用BpBinder对象的transact()方法，需要注意的是这个BpBinder对象是持有AlarmManagerService的handle值的BpBinder，而不是handle值为0的BpBinder.  
之后仍然是调用IPCThreadState::transact()方法，writeTransactionData()也都差别不大,但是注意handle,code值不一样,其中handle为AlarmManagerService对应的handle值，code则为Stub.TRANSACTION_setTime.  
然后调用waitForResponse()方法，客户端进程进入等待状态。  

而Service Server进程则被唤醒，开始执行命令，下面是IPCThreadState::executeCommand()方法的主要代码：  

    status_t IPCThreadState::executeCommand(int32_t cmd)
    {
        BBinder* obj;
        RefBase::weakref_type* refs;
        status_t result = NO_ERROR;
        
        switch (cmd) {
        
            ...

        case BR_TRANSACTION:
            {
                binder_transaction_data tr;
                result = mIn.read(&tr, sizeof(tr));
                LOG_ASSERT(result == NO_ERROR,
                    "Not enough command data for brTRANSACTION");
                if (result != NO_ERROR) break;
                
                Parcel buffer;
                buffer.ipcSetDataReference(
                    reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                    tr.data_size,
                    reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                    tr.offsets_size/sizeof(size_t), freeBuffer, this);
                
                const pid_t origPid = mCallingPid;
                const uid_t origUid = mCallingUid;
                
                mCallingPid = tr.sender_pid;
                mCallingUid = tr.sender_euid;
                
                int curPrio = getpriority(PRIO_PROCESS, mMyThreadId);
                if (gDisableBackgroundScheduling) {
                    if (curPrio > ANDROID_PRIORITY_NORMAL) {
                        // We have inherited a reduced priority from the caller, but do not
                        // want to run in that state in this process.  The driver set our
                        // priority already (though not our scheduling class), so bounce
                        // it back to the default before invoking the transaction.
                        setpriority(PRIO_PROCESS, mMyThreadId, ANDROID_PRIORITY_NORMAL);
                    }
                } else {
                    if (curPrio >= ANDROID_PRIORITY_BACKGROUND) {
                        // We want to use the inherited priority from the caller.
                        // Ensure this thread is in the background scheduling class,
                        // since the driver won't modify scheduling classes for us.
                        // The scheduling group is reset to default by the caller
                        // once this method returns after the transaction is complete.
                        androidSetThreadSchedulingGroup(mMyThreadId,
                                                        ANDROID_TGROUP_BG_NONINTERACT);
                    }
                }

                Parcel reply;
            
                if (tr.target.ptr) {
                    sp<BBinder> b((BBinder*)tr.cookie);
                    const status_t error = b->transact(tr.code, buffer, &reply, 0);
                    if (error < NO_ERROR) reply.setError(error);
                    
                } else {
                    const status_t error = the_context_object->transact(tr.code, buffer, &reply, 0);
                    if (error < NO_ERROR) reply.setError(error);
                }
                
                if ((tr.flags & TF_ONE_WAY) == 0) {
                    LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                    sendReply(reply, 0);
                } else {
                    LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
                }
                
                mCallingPid = origPid;
                mCallingUid = origUid;
            }
            break;
        
            ...

        }

        if (result != NO_ERROR) {
            mLastError = result;
        }
        
        return result;
    }

这里先将tr.cookie转化为BBinder指针.前面已经说过,与单纯的本地服务不同，tr.cookie中存放的不是类似MediaPlayerService对象指针，而是JavaBBinder对象指针，而之前分析过BBinder的transact()中会调用onTransact()方法，JavaBBinder又重写了onTransact()方法，因而此处调用的是JavaBBinder中的onTransact()方法，其代码主要如下：  

    virtual status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
    {
        JNIEnv* env = javavm_to_jnienv(mVM);

        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, (int32_t)&data, (int32_t)reply, flags);
        jthrowable excep = env->ExceptionOccurred();
        if (excep) {
            report_exception(env, excep,
                "*** Uncaught remote exception!  "
                "(Exceptions are not yet supported across processes.)");
            res = JNI_FALSE;

            /* clean up JNI local ref -- we don't return to Java code */
            env->DeleteLocalRef(excep);
        }

        //aout << "onTransact to Java code; result=" << res << endl
        //    << "Transact from " << this << " to Java code returning "
        //    << reply << ": " << *reply << endl;
        return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
    }

前面说过，JavaBBinder对象中mObject成员是AlarmManagerService对象，因而这里调用的是AlarmManagerService对象的execTransact()方法(因为AlarmManagerService继承于IAlarmManager.Stub,而IAlarmManager.Stub又继承于Binder,Binder中定义了execTransact()方法),代码如下：  

    private boolean execTransact(int code,int dataObj,int replyObj,int flags)
    {
        Parcel data=Parcel.obtain(dataObj);
        Parcel reply=Parcel.obtain(replyObj);

        boolean res;
        try{
            res=onTransact(code,data,reply,flags);
        }catch(RemoteException e){
            reply.writeException(e);
            res=true;
        }catch(RuntimeException e){
            reply.writeException(e);
            res=true;
        }
        reply.recycle();
        data.recycle();
        return res;
    }

显然，这里会调用Stub中的onTransact()方法，代码如下：  

    @Override 
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
        switch (code)
        {
        
            ...
            
            case TRANSACTION_setTime:
            {
                data.enforceInterface(DESCRIPTOR);
                long _arg0;
                _arg0 = data.readLong();
                this.setTime(_arg0);
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_setTimeZone:
            {
                data.enforceInterface(DESCRIPTOR);
                java.lang.String _arg0;
                _arg0 = data.readString();
                this.setTimeZone(_arg0);
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_remove:
            {
                data.enforceInterface(DESCRIPTOR);
                android.app.PendingIntent _arg0;
                if ((0!=data.readInt())) {
                    _arg0 = android.app.PendingIntent.CREATOR.createFromParcel(data);
                }
                else {
                    _arg0 = null;
                }
                this.remove(_arg0);
                reply.writeNoException();
                return true;
            }
        }
      return super.onTransact(code, data, reply, flags);
    }

