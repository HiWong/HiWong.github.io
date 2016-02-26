---
layout: post
title: "Android Binder机制(3) 本地服务注册过程"
date: 2016-02-27 00:27:56 +0800
comments: true
categories: android_deep_analysis
---

本博客将讲解本地服务的注册过程，为了方便大家更好地理解，选择了MediaPlayer Service作为例子。  

启动并注册MediaPlayer Service的代码在frameworks/base/media/mediaserver/main_mediaserver.cpp中，如下：  

    int main(int argc, char** argv)
    {
        sp<ProcessState>proc(ProcessState::self());
        sp<IServiceManager>sm=defaultServiceManager();
        LOGI("ServiceManager: %p",sm.get());
        AudioFlinger::instantiate();
        MediaPlayerService::instantiate();
        CameraService::instantiate();
        AudioPolicyService::instantiate();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }

表面上看，启动服务的代码异常简单，实际上只是代码封装得好，里面的调用非常复杂。下面我们将逐一剖析。  

1.sp<ProcessState>proc(ProcessState::self());  

首先我们看ProcessState::self();这个方法，如下所示<!--more-->  

    sp<ProcessState>ProcessState::self()
    {
        if(gProcess!=NULL) return gProcess;

        AutoMutex -l(gProcessMutex);
        if(gProcess==NULL) gProcess=new ProcessState;
        return gProcess;
    }

显然，gProcess是一个全局变量，由于ProcessState的构造函数为私有构造函数，所以只能采用static函数生成ProcessState实例。再来看它的构造函数，如下所示：  

    ProcessState::ProcessState()
        : mDriverFD(open_driver())
        , mVMStart(MAP_FAILED)
        , mManagesContexts(false)
        , mBinderContextCheckFunc(NULL)
        , mBinderContextUserData(NULL)
        , mThreadPoolStarted(false)
        , mThreadPoolSeq(1)
    {
        if (mDriverFD >= 0) {
            // XXX Ideally, there should be a specific define for whether we
            // have mmap (or whether we could possibly have the kernel module
            // availabla).
    #if !defined(HAVE_WIN32_IPC)
            // mmap the binder, providing a chunk of virtual address space to receive transactions.
            mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
            if (mVMStart == MAP_FAILED) {
                // *sigh*
                LOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
                close(mDriverFD);
                mDriverFD = -1;
            }
    #else
            mDriverFD = -1;
    #endif
        }
        if (mDriverFD < 0) {
            // Need to run without the driver, starting our own thread pool.
        }
    }

构造函数的实体比较简单，主要就是调用了mmap()方法，但是它在成员列表中进行了很多的操作，除了为成员变量设置初始值之外，主要就是open_driver()返回文件描述符了。  

open_driver()将打开"dev/binder",并且返回一个文件描述符(其实就是一个整型数)给mDriverFD;其代码如下：  

    static int open_driver()
    {
        if (gSingleProcess) {
            return -1;
        }

        int fd = open("/dev/binder", O_RDWR);
        if (fd >= 0) {
            fcntl(fd, F_SETFD, FD_CLOEXEC);
            int vers;
    #if defined(HAVE_ANDROID_OS)
            status_t result = ioctl(fd, BINDER_VERSION, &vers);
    #else
            status_t result = -1;
            errno = EPERM;
    #endif
            if (result == -1) {
                LOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
                close(fd);
                fd = -1;
            }
            if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
                LOGE("Binder driver protocol does not match user space protocol!");
                close(fd);
                fd = -1;
            }
    #if defined(HAVE_ANDROID_OS)
            size_t maxThreads = 15;
            result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
            if (result == -1) {
                LOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
            }
    #endif
            
        } else {
            LOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
        }
        return fd;
    }

open_drvier()的代码量虽然不少，但是其实主要就是做了两件事：第一，打开/dev/binder这个binder设备，这个是android在内核中设置的一个专门用于完成进程间通讯的虚拟设备;第二，result=ioctl(fd,BINDER_SET_MAX_THREADS,&maxThreads);的作用是通过ioctl()的方式告诉内核，这个fd支持的最大线程数是15个。  

下面再回到ProcessState的构造函数中分析mmap()，其实非常简单，就是根据返回的文件描述符，将用户空间的特定区域映射到内核空间的特定区域中。由于用户空间中的进程不能直接访问内核空间，所以只能通过内核空间的特定映射区域来访问内核空间。当调用mmap()函数时，将从0x40000000地址开始开辟一块指定大小的空间，而后调用内核的binder_mmap()函数。  

在Android系统中，内核空间以及由mmap()函数映射出的区域都事先被定义好，Android系统采用Prelinked方式预先确定各个库将被连接的地址。这些连接信息在/build/core/prelink-linux-arm.map中可以查看到，如下所示：  

    0000000 - 0xFFFFFFFF Kernel
    # 0xB0100000 - 0xBFFFFFFF Thread 0 Stack
    # 0xB0000000 - 0xB00FFFFF Linker
    # 0xA0000000 - 0xBFFFFFFF Prelinked System Libraries
    # 0x90000000 - 0x9FFFFFFF Prelinked App Libraries
    # 0x80000000 - 0x8FFFFFFF Non-prelinked Libraries
    # 0x40000000 - 0x7FFFFFFF mmap'd stuff
    # 0x10000000 - 0x3FFFFFFF Thread Stacks
    # 0x00000000 - 0x0FFFFFFF .text / .data / heap

其中mmap'd stuff即为mmap()函数的起始映射地址。  
至此，main()函数中的第一行代码就分析完了，总结一下，主要做了以下两件事：

+ 打开/dev/binder设备，并且根据返回的文件描述符将用户空间的特定区域映射到内核空间的特定区域中;
+ 新建了一个ProcessState对象

整个创建过程可以用一个简单的示意图表示如下：  

![ProcessState](http://7xn1yt.com1.z0.glb.clouddn.com/ProcessState.png)


2.sp<IServiceManager>sm=defaultServiceManager();  

其中defaultServiceManager()的代码如下：  

    sp<IServiceManager>defaultServiceManager()
    {
        if(gDefaultServiceManager!=NULL) return gDefaultServiceManager;

        {
            AutoMutex _l(gDefaultServiceManagerLock);
            if(gDefaultServiceManager==NULL){
                gDefaultServiceManager=interface_cast<IServiceManager>(
                    ProcessState::self()->getContextObject(NULL));
            }
        }

        return gDefaultServiceManager;
    }

显然这里采用了单例设计模式，gDefaultServiceManager只会创建一次，而ProcessState::self()在前面已经分析过，下面看ProcessState::getContextObject()方法：  

    sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& caller)
    {
        if (supportsProcesses()) {
            return getStrongProxyForHandle(0);
        } else {
            return getContextObject(String16("default"), caller);
        }
    }

在真机上supportsProcesses()为true，所以进入getStrongProxyForHandle()方法：  

    sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
    {
        sp<IBinder> result;

        AutoMutex _l(mLock);

        handle_entry* e = lookupHandleLocked(handle);

        if (e != NULL) {
            // We need to create a new BpBinder if there isn't currently one, OR we
            // are unable to acquire a weak reference on this current one.  See comment
            // in getWeakProxyForHandle() for more info about this.
            IBinder* b = e->binder;
            if (b == NULL || !e->refs->attemptIncWeak(this)) {
                b = new BpBinder(handle); 
                e->binder = b;
                if (b) e->refs = b->getWeakRefs();
                result = b;
            } else {
                // This little bit of nastyness is to allow us to add a primary
                // reference to the remote proxy when this team doesn't have one
                // but another team is sending the handle to us.
                result.force_set(b);
                e->refs->decWeak(this);
            }
        }

        return result;
    }

首先分析被调用的lookupHandleLocked()方法：  

    ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
    {
        const size_t N=mHandleToObject.size();
        if (N <= (size_t)handle) {
            handle_entry e;
            e.binder = NULL;
            e.refs = NULL;
            status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
            if (err < NO_ERROR) return NULL;
        }
        return &mHandleToObject.editItemAt(handle);
    }

其中mHandleToObject是一个Vector<handle_entry>对象，Vector<Typename>相关的资料在C++ STL中的资料可查看的，读者可将其简单地理解成handle_entry对象组成的序列，由于传入的handle==0,所以N<=(size_t)handle肯定成立 ，从而创建一个handle_entry对象，创建完之后插入到mHandleToObject中，显然此对象的binder成员为NULL。再回到ProcessState::getStrongProxyForHandle()方法中，显然此时会创立一个BpBinder对象，这一点从注释中也可以看到。  

所以，gDefaultServiceManager=interface_cast<IServiceManger>(ProcessState::self()->getContextObject(NULL));在这里可以等价为：  

    gDefaultServiceManager=interface_cast<IServiceManager>(new BpBinder(0));

下面我们看一下BpBinder的构造函数：  

    BpBinder::BpBinder(int32_t handle)
        : mHandle(handle)
        , mAlive(1)
        , mObitsSent(0)
        , mObituaries(NULL)
    {
        LOGV("Creating BpBinder %p handle %d\n",this, mHandle);

        extendObjectLifetime(OBJECT_LIFETIME_WEAK);
        IPCThreadState::self()->incWeakHandle(handle);
    }

成员初始化列表比较简单，下面重点讲一下IPCThreadState::self()->incWeakHandle(handle),下面是IPCThreadState::self()的代码：  

    IPCThreadState* IPCThreadState::self()
    {
        if (gHaveTLS) {
    restart:
            const pthread_key_t k = gTLS;
            IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
            if (st) return st;
            return new IPCThreadState;
        }
        
        if (gShutdown) return NULL;
        
        pthread_mutex_lock(&gTLSMutex);
        if (!gHaveTLS) {
            if (pthread_key_create(&gTLS, threadDestructor) != 0) {
                pthread_mutex_unlock(&gTLSMutex);
                return NULL;
            }
            gHaveTLS = true;
        }
        pthread_mutex_unlock(&gTLSMutex);
        goto restart;
    }

显然IPCThreadState
注意gHaveTLS的意思是是否含有Thread Local Storage，首次进入时没有，所以代码走到下面，pthread_key_create(&gTLS,threadDestructor)的主要作用就是新建一个TLS key并且保存相应的析构函数指针。显然，后面再进入IPCThreadState::self()这个函数时，就只需要通过pthread_getspecific(k)获取相应的TLS key即可。  

下面看一下IPCThreadState的构造方法：  

    IPCThreadState::IPCThreadState()
        :mProcess(ProcessState::self()),mMyThreadId(androidGetTid())
    {
        pthread_setspecific(gTLS,this);
        clearCaller();
        mIn.setDataCapacity(256);
        mOut.setDataCapacity(256);
    }

首先是在成员初始化列表中为mProcess和mMyThreadId赋值。mIn,mOut是用于与Binder Driver通信的Parcel对象。然后pthread_setspecific()方法如下：  

    int pthread_setspecific(pthread_key_t key,const void *ptr)
    {
        int err=EINVAL;
        tlsmap_t* map;

        if(TLSMAP_VALIDATE_KEY(key)){
            /*check that we're trying to set data for an allocated key*/
            map=tlsmap_lock();
            if(tlsmap_test(map,key)){
                ((uint32_t*)__get_tls())[key]=(uint32_t)ptr;
                err=0;
            }
            tlsmap_unlock(map);
        }
        return err;
    }

显然thread_setspecific(gTLS,this);的作用就是为map中key为gTLS赋值为当前IPCThreadState对象。

至此，总结一下，在defaultServiceManager()中我们主要做了以下几个工作：  

+ 创建了IPCThreadState，并且为其创建了TLS;

+ 创建了handle值为0的BpBinder

那 gDefaultServiceManager=interface_cast<IServiceManager>(new BpBinder(0)); 是如何实现的呢？  

先看一下interface_cast模板函数：  

    template<typename INTERFACE>
    inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
    {
        return INTERFACE::asInterface(obj);
    }

但是我们会发现asInterface()的代码是不存在的，不过有如下两段宏：  

    DECLARE_META_INTERFACE(ServiceManager);
    IMPLEMENT_META_INTERFACE(ServiceManager,"android.os.IServiceManager")

其中前者是宏定义，后者是宏实现，将IMPLEMENT_META_INTERFACE宏扩展后，得到如下asInterface()的代码：  

    sp<IServiceManager> IServiceManager::asInterface(const sp<IBinder>& obj)
    {
        sp<IServiceManager>intr;
        if(obj!=NULL){
            intr=static_cast<IServiceManager*>(
                obj->queryLocalInterface(IServiceManager::descriptor).get());
                if(intr==NULL){
                    intr=new BpServiceManager(obj);
                }
        }
    }

IBinder的queryLocalInterface()函数将根据obj是BBinder或BpBinder而采取不同的行为动作，当参数obj是BBinder对象时，转换类型为服务对象;当参数obj是BpBinder对象时，则返回NULL.显然，这里返回的是NULL.所以会新建一个BpServiceManager对象。下面我们看一下BpServiceManager的构造函数:  

    BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl)
    {
    }

非常简单，就是将传入的BpBinder对象传给BpInterface<IServiceManager>完成构造，而BpInterface的构造函数如下：  

    template<typename INTERFACE>
    inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote)
        : BpRefBase(remote)
    {
    }

这里又再一次将BpBinder对象传递给了BpRefBase，再看一下BpRefBase的构造函数：  

    BpRefBase::BpRefBase(const sp<IBinder>& o)
        : mRemote(o.get()), mRefs(NULL), mState(0)
    {
        extendObjectLifetime(OBJECT_LIFETIME_WEAK);

        if(mRemote){
            mRemote->incStrong(this);
            mRefs=mReote->createWeak(this);
        }
    }

显然，这里将BpBinder对象传递给了BpBinder对象传递到了mRemote，其中o.get()是模板类sp中的方法，sp是Google定义的用于处理指针的模板类，可将sp理解为Strong Pointer，与之相对的是wp(Weak Pointer),此处不展开讨论。
这里BpServiceManager,BpInterface,BpRefBase的继承关系如下：  

![BpRefBase](http://7xn1yt.com1.z0.glb.clouddn.com/BpRefBase.png)

至此，sp<IServiceManager>sm=defaultServiceManager();就分析完毕，返回的是一个BpServiceManager对象,而且该BpServiceManager对象中的mRemote对象是handle值为0的BpBinder对象;  

3.MediaPlayerService::instantiate();  

该方法的代码如下：  

    void MediaPlayerService::instantiate(){
        defaultServiceManager()->addService(
            String16("media.player"),new MediaPlayerService());
    }

3.1 MediaPlayerService  

首先看一下MediaPlayerService这个类,它的构造函数非常简单，就不展开说了。关键是注意到MediaPlayerService继承自BnMediaPlayerService,BnMediaPlayerSerivce中重写了virtual函数onTransact(),如下所示：  

status_t BnMediaPlayerService::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case CREATE_URL: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            pid_t pid = data.readInt32();
            sp<IMediaPlayerClient> client =
                interface_cast<IMediaPlayerClient>(data.readStrongBinder());
            const char* url = data.readCString();

            KeyedVector<String8, String8> headers;
            int32_t numHeaders = data.readInt32();
            for (int i = 0; i < numHeaders; ++i) {
                String8 key = data.readString8();
                String8 value = data.readString8();
                headers.add(key, value);
            }

            sp<IMediaPlayer> player = create(
                    pid, client, url, numHeaders > 0 ? &headers : NULL);

            reply->writeStrongBinder(player->asBinder());
            return NO_ERROR;
        } break;
        case CREATE_FD: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            pid_t pid = data.readInt32();
            sp<IMediaPlayerClient> client = interface_cast<IMediaPlayerClient>(data.readStrongBinder());
            int fd = dup(data.readFileDescriptor());
            int64_t offset = data.readInt64();
            int64_t length = data.readInt64();
            sp<IMediaPlayer> player = create(pid, client, fd, offset, length);
            reply->writeStrongBinder(player->asBinder());
            return NO_ERROR;
        } break;
        case DECODE_URL: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            const char* url = data.readCString();
            uint32_t sampleRate;
            int numChannels;
            int format;
            sp<IMemory> player = decode(url, &sampleRate, &numChannels, &format);
            reply->writeInt32(sampleRate);
            reply->writeInt32(numChannels);
            reply->writeInt32(format);
            reply->writeStrongBinder(player->asBinder());
            return NO_ERROR;
        } break;
        case DECODE_FD: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            int fd = dup(data.readFileDescriptor());
            int64_t offset = data.readInt64();
            int64_t length = data.readInt64();
            uint32_t sampleRate;
            int numChannels;
            int format;
            sp<IMemory> player = decode(fd, offset, length, &sampleRate, &numChannels, &format);
            reply->writeInt32(sampleRate);
            reply->writeInt32(numChannels);
            reply->writeInt32(format);
            reply->writeStrongBinder(player->asBinder());
            return NO_ERROR;
        } break;
        case SNOOP: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            sp<IMemory> snooped_audio = snoop();
            reply->writeStrongBinder(snooped_audio->asBinder());
            return NO_ERROR;
        } break;
        case CREATE_MEDIA_RECORDER: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            pid_t pid = data.readInt32();
            sp<IMediaRecorder> recorder = createMediaRecorder(pid);
            reply->writeStrongBinder(recorder->asBinder());
            return NO_ERROR;
        } break;
        case CREATE_METADATA_RETRIEVER: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            pid_t pid = data.readInt32();
            sp<IMediaMetadataRetriever> retriever = createMetadataRetriever(pid);
            reply->writeStrongBinder(retriever->asBinder());
            return NO_ERROR;
        } break;
        case GET_OMX: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            sp<IOMX> omx = getOMX();
            reply->writeStrongBinder(omx->asBinder());
            return NO_ERROR;
        } break;
        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}

显然是根据不同的命令(code值)进行相应的回调操作。而BnMediaPlayerService又继承自BnInterface<IMediaPlayerService>，因而可作出如下的UML图:  

![mediaplayerservice_uml](http://7xn1yt.com1.z0.glb.clouddn.com/mediaplayerservice_uml.png)

3.2 addService()  

再回到MediaPlayerService::instantiate()这个方法中，defaultServiceManager()返回的是全局变量gDefaultServiceManager，也就是刚刚创建的BpServiceManager，而BpServiceManager::addService()方法的代码如下：

    virtual status_t addService(const String16& name,const sp<IBinder>& service)
    {
        Parcel data,reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        data.writeStrongBinder(service);
        status_t err=remote()->transact(ADD_SERVICE_TRANSACTION,data,&reply);
        return err==NO_ERROR ? reply.readInt32() : err;
    }

其中service是刚刚创建的MediaPlayerService对象。Parcel对象data的作用是保存传入的数据，在这里data中保存的数据如下所示：

![Parcel_data](http://7xn1yt.com1.z0.glb.clouddn.com/Parcel_data.png)

至于flat_binder_object会在后面说明。  

3.2.1 writeStrongBinder()  

reply的作用很明显，是用于保存返回的数据的。下面进入Parcel::writeStrongBinder()方法：  

    status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
    {
        return flatten_binder(ProcessState::self(), val, this);
    }

前面分析过，ProcessState::self()返回的是它的单例对象，而此处的val其实是MediaPlayerService对象，由于它间接继承于BBinder，而BBinder继承于IBinder，所以可作用IBinder对象使用。下面进入flatten_binder()方法：  

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

由于当前的binder其实是MediaPlayerService实例，而MediaPlayerService间接继承自BBinder，而BBinder::localBinder()返回的是this，所以local不为NULL,从而执行else部分的代码。下面看一下flat_binder_object的数据结构：  

    struct flat_binder_object{
        unsigned long type;
        unsigned long flags;
        union{
            void*binder;
            signed long handle;
        };
        void*cookie;
    }

所以执行完else部分的代码之后,其type值为BINDER_TYPE_BINDER,binder成员值则为MediaPlayerService对象的弱引用,cookie则指向MediaPlayerService对象。然后，调用finish_flatten_binder()函数，将flat_binder_object对象保存到data中。  

至于finish_flatten_binder(binder,obj,out);这个语句，展开来讲的话非常长，所以将它单独放在一篇博客中进行分析，博客链接为。  

**至此，data.writeStrongBinder(service);就分析完毕。下面进行remote()->transact(ADD_SERVICE_TRANSACTION,data,&reply)的讲解。**

3.2.3 IPCThreadState::self()->transact()  

再回到BpServiceManager::addService()方法中，前面讲过,remote()返回的是handle值为0的BpBinder对象，所以这里的remote()->transact(ADD_SERVICE_TRANSACTION,data,&reply)本质上是调用BpBinder的transact()方法，其代码如下：  

    status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        status_t status=IPCThreadState::self()->transact(
            mHandle,code,data,reply,flags);
        return status;
    }

其中code值为ADD_SERVICE_TRANSACTION,data为上面分析过的Parcel对象引用,reply用于放置返回值，flags为默认值0;
下面进入IPCThreadState::transact()方法，其主要代码如下：  

    status_t IPCThreadState::transact(int32_t handle,uint32_t code,const Parcel& data,
        Parcel* reply,uint32_t flags)
    {
        status_t err=data.errorCheck();
        flags|=TF_ACCEPT_FDS;
        err=writeTransactionData(BC_TRANSACTION,flags,
            handle,code,data,NULL);
        ...
        err=waitForResponse(reply);
        ...
        return err;
    }

3.2.3.1 writeTransactionData()分析  

下面是writeTransactionData()方法的代码如下：  

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
            tr.flags |= TF_STATUS_CODE;
            *statusBuffer = err;
            tr.data_size = sizeof(status_t);
            tr.data.ptr.buffer = statusBuffer;
            tr.offsets_size = 0;
            tr.data.ptr.offsets = NULL;
        } else {
            return (mLastError = err);
        }
        
        mOut.writeInt32(cmd);
        mOut.write(&tr, sizeof(tr));
        
        return NO_ERROR;
    }

首先我们看一下binder_transaction_data的定义：  

    struct binder_transaction_data{
        uninon{
            size_t handle;
            void*ptr;
        }target;
        void*cookie;
        unsigned int code;
        unsigned int flags;
        pid_t sender_pid;
        uid_t sender_euid;
        size_t data_size;
        size_t offsets_size;
        union{
            struct{
                const void*buffer;
                const void*offsets;
            }ptr;
            uint8_t buf[8];
        }data;
    };

所以writeTransactionData()方法就好理解了:

+ 首先将handle(此处值为0)赋值给tr.target.handle;
+ 然后将code值(ADD_SERVICE_TRANSACTION)赋给tr.code;
+ 之后将binderFlags(值为0)赋值给tr.flags;
+ data_size中保存的是data.ipcDataSize()的返回值，ipcDataSize()方法如下：  

    size_t Parcel::ipcDataSize() const
    {
        return (mDataSize>mDataPos?mDataSize:mDataPos);
    }

由博客[Android Binder机制(3) Parcel类分析](http://blog.imallen.wang/)的分析可知,写入每个字符串之前会先写入字符串长度(32位整型数，所以是4字节)，而且写入的字符串中每个字符占2个字节(而不是一个),再加上flat_binder_object对象，所以此处data_size=4+26*2+4+12*2+16=100
此时binder_transaction_data对象中的内容如下图所示：  

![binder_transaction_data](http://7xn1yt.com1.z0.glb.clouddn.com/binder_transaction_data.png)

+ buffer保存着data(送信Parcel)成员变量mData的指针
+ offset_size保存着数据4,它是data(送信Parcel)成员变量mObject中的数据大小
+ offsets保存着data成员变量mObject的指针

3.2.3.2 IPCThreadState::waitForResponse()  

该方法的代码如下：  

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
            
            IF_LOG_COMMANDS() {
                alog << "Processing waitForResponse Command: "
                    << getReturnString(cmd) << endl;
            }

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

显然，最先调用了talkWithDriver()方法，然后从mIn中读出命令，再根据具体的命令采取相应的动作。  
下面是talkWithDriver()的代码(注意其参数doReceive默认值为true)：  




















