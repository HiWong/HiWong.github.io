---
layout: post
title: "Android Binder机制分析(4) Parcel类分析"
date: 2016-02-27 00:33:34 +0800
comments: true
categories: android_deep_analysis
---


引言  

在上一篇Blog中，在分析服务注册过程时，往data(Parcel对象)变量写入数据时，有这样的调用路径：  

BpServiceManager::addService()-->Parcel::writeStrongBinder()-->flatten_binder()-->finish_flatten_binder()

由于finish_flatten_binder()方法中涉及到的东西太多，在上一篇博客就没有展开来讲。这篇博客将详细分析数据是如何写入到data中的。  

下面是Parcel类的定义：<!--more-->  
``` cpp Parcel.h
class Parcel
{
public:
                        Parcel();
                        ~Parcel();
    
    const uint8_t*      data() const;
    size_t              dataSize() const;
    size_t              dataAvail() const;
    size_t              dataPosition() const;
    size_t              dataCapacity() const;
    
    status_t            setDataSize(size_t size);
    void                setDataPosition(size_t pos) const;
    status_t            setDataCapacity(size_t size);
    
    status_t            setData(const uint8_t* buffer, size_t len);

    status_t            appendFrom(Parcel *parcel, size_t start, size_t len);

    bool                hasFileDescriptors() const;

    status_t            writeInterfaceToken(const String16& interface);
    bool                enforceInterface(const String16& interface) const;
    bool                checkInterface(IBinder*) const;    

    void                freeData();

    const size_t*       objects() const;
    size_t              objectsCount() const;
    
    status_t            errorCheck() const;
    void                setError(status_t err);
    
    status_t            write(const void* data, size_t len);
    void*               writeInplace(size_t len);
    status_t            writeUnpadded(const void* data, size_t len);
    status_t            writeInt32(int32_t val);
    status_t            writeInt64(int64_t val);
    status_t            writeFloat(float val);
    status_t            writeDouble(double val);
    status_t            writeIntPtr(intptr_t val);
    status_t            writeCString(const char* str);
    status_t            writeString8(const String8& str);
    status_t            writeString16(const String16& str);
    status_t            writeString16(const char16_t* str, size_t len);
    status_t            writeStrongBinder(const sp<IBinder>& val);
    status_t            writeWeakBinder(const wp<IBinder>& val);
    status_t            write(const Flattenable& val);

    // Place a native_handle into the parcel (the native_handle's file-
    // descriptors are dup'ed, so it is safe to delete the native_handle
    // when this function returns). 
    // Doesn't take ownership of the native_handle.
    status_t            writeNativeHandle(const native_handle* handle);
    
    // Place a file descriptor into the parcel.  The given fd must remain
    // valid for the lifetime of the parcel.
    status_t            writeFileDescriptor(int fd);
    
    // Place a file descriptor into the parcel.  A dup of the fd is made, which
    // will be closed once the parcel is destroyed.
    status_t            writeDupFileDescriptor(int fd);
    
    status_t            writeObject(const flat_binder_object& val, bool nullMetaData);

    void                remove(size_t start, size_t amt);
    
    status_t            read(void* outData, size_t len) const;
    const void*         readInplace(size_t len) const;
    int32_t             readInt32() const;
    status_t            readInt32(int32_t *pArg) const;
    int64_t             readInt64() const;
    status_t            readInt64(int64_t *pArg) const;
    float               readFloat() const;
    status_t            readFloat(float *pArg) const;
    double              readDouble() const;
    status_t            readDouble(double *pArg) const;
    intptr_t            readIntPtr() const;
    status_t            readIntPtr(intptr_t *pArg) const;

    const char*         readCString() const;
    String8             readString8() const;
    String16            readString16() const;
    const char16_t*     readString16Inplace(size_t* outLen) const;
    sp<IBinder>         readStrongBinder() const;
    wp<IBinder>         readWeakBinder() const;
    status_t            read(Flattenable& val) const;
    
    // Retrieve native_handle from the parcel. This returns a copy of the
    // parcel's native_handle (the caller takes ownership). The caller
    // must free the native_handle with native_handle_close() and 
    // native_handle_delete().
    native_handle*     readNativeHandle() const;

    
    // Retrieve a file descriptor from the parcel.  This returns the raw fd
    // in the parcel, which you do not own -- use dup() to get your own copy.
    int                 readFileDescriptor() const;
    
    const flat_binder_object* readObject(bool nullMetaData) const;

    // Explicitly close all file descriptors in the parcel.
    void                closeFileDescriptors();
    
    typedef void        (*release_func)(Parcel* parcel,
                                        const uint8_t* data, size_t dataSize,
                                        const size_t* objects, size_t objectsSize,
                                        void* cookie);
                        
    const uint8_t*      ipcData() const;
    size_t              ipcDataSize() const;
    const size_t*       ipcObjects() const;
    size_t              ipcObjectsCount() const;
    void                ipcSetDataReference(const uint8_t* data, size_t dataSize,
                                            const size_t* objects, size_t objectsCount,
                                            release_func relFunc, void* relCookie);
    
    void                print(TextOutput& to, uint32_t flags = 0) const;
        
private:
                        Parcel(const Parcel& o);
    Parcel&             operator=(const Parcel& o);
    
    status_t            finishWrite(size_t len);
    void                releaseObjects();
    void                acquireObjects();
    status_t            growData(size_t len);
    status_t            restartWrite(size_t desired);
    status_t            continueWrite(size_t desired);
    void                freeDataNoInit();
    void                initState();
    void                scanForFds() const;
                        
    template<class T>
    status_t            readAligned(T *pArg) const;

    template<class T>   T readAligned() const;

    template<class T>
    status_t            writeAligned(T val);

    status_t            mError;
    uint8_t*            mData;
    size_t              mDataSize;
    size_t              mDataCapacity;
    mutable size_t      mDataPos;
    size_t*             mObjects;
    size_t              mObjectsSize;
    size_t              mObjectsCapacity;
    mutable size_t      mNextObjectHint;

    mutable bool        mFdsKnown;
    mutable bool        mHasFds;
    
    release_func        mOwner;
    void*               mOwnerCookie;
};
```
虽然public方法有很多，但是我们先只要注意它的成员变量即可，特别是mData,mDataSize,mDataCapacity,mDataPos,mObjects,mObjectsSize,mObjectsCapacity这几个成员变量。  

另外，为了方便大家理解，先说1个结论：对于普通数据，使用mData进行储存;对于IBinder类型的数据以及FileDescriptor使用的是mObjects;  


1.Parcel类的初始化  

在C++中，类的初始化非常重要，可能在成员初始化列表中就进行非常多的操作。所以首先要看它的构造函数：  
``` cpp 
Parcel::Parcel()
{
    initState();
}
```
出乎意料地是，Parcel类的构造方法异常简单，就是调用initState()方法，下面是initState()的代码：  
``` cpp 
void Parcel::initState()
{
    mError = NO_ERROR;
    mData = 0;
    mDataSize = 0;
    mDataCapacity = 0;
    mDataPos = 0;
    LOGV("initState Setting data size of %p to %d\n", this, mDataSize);
    LOGV("initState Setting data pos of %p to %d\n", this, mDataPos);
    mObjects = NULL;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mHasFds = false;
    mFdsKnown = true;
    mOwner = NULL;
}
```
可以发现，initState()函数主要就是对成员变量初始化。而mDataSize,mDataCapacity,mDataPos,mObjectsSize,mObjectsCapacity都为0，另外指针都初始化为NULL.注意这些成员的初始值很重要，因为后面会用到。  

2.finish_flatten_binder()  

由于在上一篇Blog中已经分析过Parcel::writeStrongBinder()和flatten_binder()函数，所以这里直接分析finish_flatten_binder()方法了。该方法的代码如下：  
``` cpp
inline static status_t finish_flatten_binder(const sp<IBinder>& binder,const flat_binder_object& flat,Parcel* out)
{
    return out->writeObject(flat,false);
}
```
竟然又是调用，这个嵌套太多了,而且binder参数没有再被用到的话，其实就可以不传递过来了。下面看Parcel::writeObject()方法的代码：  
``` cpp
status_t Parcel::writeObject(const flat_binder_object& val, bool nullMetaData)
{
    const bool enoughData = (mDataPos+sizeof(val)) <= mDataCapacity;
    const bool enoughObjects = mObjectsSize < mObjectsCapacity;
    if (enoughData && enoughObjects) {
restart_write:
         //code_1
        *reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val;
        
        // Need to write meta-data?
        //code_2
        if (nullMetaData || val.binder != NULL) {
            mObjects[mObjectsSize] = mDataPos;
            acquire_object(ProcessState::self(), val, this);
            mObjectsSize++;
        }
        
        // remember if it's a file descriptor
        //code_3
        if (val.type == BINDER_TYPE_FD) {
            mHasFds = mFdsKnown = true;
        }
        //code_4
        return finishWrite(sizeof(flat_binder_object));
    }

    //code_5
    if (!enoughData) {
        const status_t err = growData(sizeof(val));
        if (err != NO_ERROR) return err;
    }
    //code_6
    if (!enoughObjects) {
        size_t newSize = ((mObjectsSize+2)*3)/2;
        size_t* objects = (size_t*)realloc(mObjects, newSize*sizeof(size_t));
        if (objects == NULL) return NO_MEMORY;
        mObjects = objects;
        mObjectsCapacity = newSize;
    }
    
    goto restart_write;
}
```
由于从Parcel data;定义之后，一直到这里，除了写入接口描述符和服务名称之外，并没有改变data的其他成员变量，所以mDataPos,mDataCapacity,mObjectsSize,mObjectsCapacity仍然是初始值0,所以enoughData,enoughObjects均为false.  

所以先执行code_5和code_6处的代码，先看growData()方法的代码：  
``` cpp
status_t Parcel::growData(size_t len)
{
    size_t newSize=((mDataSize+len)*3)/2;
    return (newSize<=mDataSize)
            ? (status_t) NO_MEMORY
            : continueWrite(newSize);
}
```
其中len是flat_binder_object的大小，**显然这里无论如何都不会出现newSize<=mDataSize的情况，所以这里是个很明显的bug，存在着内存泄露的风险。**
下面我们看一下continueWrite(newSize)的代码，注意此时传入的参数值是(mDataSize+len)*3/2：  
``` cpp
status_t Parcel::continueWrite(size_t desired)
{
    // If shrinking, first adjust for any objects that appear
    // after the new data size.
    size_t objectsSize = mObjectsSize;
    //code_1
    if (desired < mDataSize) {
        if (desired == 0) {
            objectsSize = 0;
        } else {
            while (objectsSize > 0) {
                if (mObjects[objectsSize-1] < desired)
                    break;
                objectsSize--;
            }
        }
    }
    //code_2
    if (mOwner) {
        // If the size is going to zero, just release the owner's data.
        if (desired == 0) {
            freeData();
            return NO_ERROR;
        }

        // If there is a different owner, we need to take
        // posession.
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
            mError = NO_MEMORY;
            return NO_MEMORY;
        }
        size_t* objects = NULL;
        
        if (objectsSize) {
            objects = (size_t*)malloc(objectsSize*sizeof(size_t));
            if (!objects) {
                mError = NO_MEMORY;
                return NO_MEMORY;
            }

            // Little hack to only acquire references on objects
            // we will be keeping.
            size_t oldObjectsSize = mObjectsSize;
            mObjectsSize = objectsSize;
            acquireObjects();
            mObjectsSize = oldObjectsSize;
        }
        
        if (mData) {
            memcpy(data, mData, mDataSize < desired ? mDataSize : desired);
        }
        if (objects && mObjects) {
            memcpy(objects, mObjects, objectsSize*sizeof(size_t));
        }
        //LOGI("Freeing data ref of %p (pid=%d)\n", this, getpid());
        mOwner(this, mData, mDataSize, mObjects, mObjectsSize, mOwnerCookie);
        mOwner = NULL;

        mData = data;
        mObjects = objects;
        mDataSize = (mDataSize < desired) ? mDataSize : desired;
        LOGV("continueWrite Setting data size of %p to %d\n", this, mDataSize);
        mDataCapacity = desired;
        mObjectsSize = mObjectsCapacity = objectsSize;
        mNextObjectHint = 0;

    } else if (mData) {    //code_3
        if (objectsSize < mObjectsSize) {
            // Need to release refs on any objects we are dropping.
            const sp<ProcessState> proc(ProcessState::self());
            for (size_t i=objectsSize; i<mObjectsSize; i++) {
                const flat_binder_object* flat
                    = reinterpret_cast<flat_binder_object*>(mData+mObjects[i]);
                if (flat->type == BINDER_TYPE_FD) {
                    // will need to rescan because we may have lopped off the only FDs
                    mFdsKnown = false;
                }
                release_object(proc, *flat, this);
            }
            size_t* objects =
                (size_t*)realloc(mObjects, objectsSize*sizeof(size_t));
            if (objects) {
                mObjects = objects;
            }
            mObjectsSize = objectsSize;
            mNextObjectHint = 0;
        }

        // We own the data, so we can just do a realloc().
        if (desired > mDataCapacity) {
            uint8_t* data = (uint8_t*)realloc(mData, desired);
            if (data) {
                mData = data;
                mDataCapacity = desired;
            } else if (desired > mDataCapacity) {
                mError = NO_MEMORY;
                return NO_MEMORY;
            }
        } else {
            mDataSize = desired;
            LOGV("continueWrite Setting data size of %p to %d\n", this, mDataSize);
            if (mDataPos > desired) {
                mDataPos = desired;
                LOGV("continueWrite Setting data pos of %p to %d\n", this, mDataPos);
            }
        }
        
    } else {   //code_4
        // This is the first data.  Easy!
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
            mError = NO_MEMORY;
            return NO_MEMORY;
        }
        
        if(!(mDataCapacity == 0 && mObjects == NULL
             && mObjectsCapacity == 0)) {
            LOGE("continueWrite: %d/%p/%d/%d", mDataCapacity, mObjects, mObjectsCapacity, desired);
        }
        
        mData = data;
        mDataSize = mDataPos = 0;
        LOGV("continueWrite Setting data size of %p to %d\n", this, mDataSize);
        LOGV("continueWrite Setting data pos of %p to %d\n", this, mDataPos);
        mDataCapacity = desired;
    }

    return NO_ERROR;
}
```
其实这个方法写得挺糟糕的，有那么多分支，应该增加几个工具函数，这样一个函数的代码量就不会那么大了，阅读代码就会方便很多。  
由于传入的desired值为(mDataSize+size(val))*3/2，肯定大于mDataSize，再加上mOwner和mData仍然为NULL，所以这里不会执行code_1,code_2和code_3这3部分代码。只需要看code_4这个分支下的代码。  

这段代码非常简单，就是先调用malloc()方法分配一块(mDataSize+size(val))*3/2大小的内存，然后让mData指向该内存，并且将mDataCapacity的值修改为分配内存的大小。  

到这里可以归纳一下,growData()方法只是分配了一块比flat_binder_object对象大一些的内存，但是暂时还没有将数据放进去。  

=============================================================

再回到Parcela::writeObject()方法中，下面执行的仍然是从堆上分配内存,分配好之后让mObjects指向内存的起始位置，并且mObjectsCapacity赋值为刚刚分配的内存大小。另外，由于mObjectsSize初始值为0，所以这里其实newSize==3,从而分配的内存大小其实是3*sizeof(size_t);其中size_t是typedef unsigned long size_t,所以sizeof(size_t)的结果为4，从而这里最终是分配了12个字节的内存大小。  

**接着是goto restart_write，再次吐槽一句，Android Framework的架构虽然很好，但是有些代码质量堪忧，像这样的goto语句在很多个地方出现过。还有像interpret_cast这样很容易导致程序crash的转化其实也应该尽量避免的。**  

*reinterpret_cast<flat_binder_object*>(mData+mDataPos)=val;的作用是将从mData+mDataPos位置处开始的内存内容填充为flat_binder_object对象val的值;  

虽然nullMetaData为false，但是val.binder不为NULL，所以下面会给mObjects数组赋值，此时mObjectsSize==0，所以mObjects数组中第一个元素为mDataPos，下面进入acquire_object()方法：  
``` cpp 
void acquire_object(const sp<ProcessState>& proc,
const flat_binder_object& obj, const void* who)
{
    switch (obj.type) {
        case BINDER_TYPE_BINDER:
            if (obj.binder) {
                LOG_REFS("Parcel %p acquiring reference on local %p", who, obj.cookie);
                static_cast<IBinder*>(obj.cookie)->incStrong(who);
            }
            return;
        case BINDER_TYPE_WEAK_BINDER:
            if (obj.binder)
                static_cast<RefBase::weakref_type*>(obj.binder)->incWeak(who);
            return;
        case BINDER_TYPE_HANDLE: {
            const sp<IBinder> b = proc->getStrongProxyForHandle(obj.handle);
            if (b != NULL) {
                LOG_REFS("Parcel %p acquiring reference on remote %p", who, b.get());
                b->incStrong(who);
            }
            return;
        }
        case BINDER_TYPE_WEAK_HANDLE: {
            const wp<IBinder> b = proc->getWeakProxyForHandle(obj.handle);
            if (b != NULL) b.get_refs()->incWeak(who);
            return;
        }
        case BINDER_TYPE_FD: {
            // intentionally blank -- nothing to do to acquire this, but we do
            // recognize it as a legitimate object type.
            return;
        }
    }

    LOGD("Invalid object type 0x%08lx", obj.type);
}
```
注意到传入的参数分别是:ProcessState::self(),flat_binder_object对象以及Parcel对象,由于obj.type为BINDER_TYPE_BINDER，所以obj的cookie对象，其实就是之前创建的MediaPlayerService对象执行incStrong()函数，这里涉及到指针的引用问题，后面会有专门的博客进行深入讲解，这里可以将其简单地理解成为Parcel对象增加一次引用。  

之后再回到Parcel::writeObject()方法中，mObjectsSize执行自加操作之后变为1.

==========================================================================================

由于val.type值是BINDER_TYPE_BINDER，所以下面进入到Parcel::finishWrite()方法中,注意传入的参数sizeof(flat_binder_object)值为16：  
``` cpp
status_t Parcel::finishWrite(size_t len)
{
    //printf("Finish write of %d\n", len);
    mDataPos += len;
    LOGV("finishWrite Setting data pos of %p to %d\n", this, mDataPos);
    if (mDataPos > mDataSize) {
        mDataSize = mDataPos;
        LOGV("finishWrite Setting data size of %p to %d\n", this, mDataSize);
    }
    //printf("New pos=%d, size=%d\n", mDataPos, mDataSize);
    return NO_ERROR;
}
```
这个方法异常简单，就是让mDataPos增加到刚刚写入数据的末尾，并且进行一个判断，如果mDataPos>mDataSize的话，就将mDataSize=mDataPos;而实际上mDataSize在赋值前还是0,所以会进行这个赋值操作，因此我们可以知道，其实mDataSize是记录当前mData中写入数据的大小。  

到这里，我们就将flat_binder_object这个对象写入到Parcel对象中的mData指向的内存中。  











































