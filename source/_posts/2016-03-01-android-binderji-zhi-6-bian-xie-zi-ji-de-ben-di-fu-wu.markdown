---
layout: post
title: "Android Binder机制(6) 编写自己的本地服务"
date: 2016-03-01 11:59:47 +0800
comments: true
categories: android_deep_analysis
---



前面几篇博客中系统地介绍了本地服务的注册、检索以及使用过程。这篇博客我们将完成一个属于自己的本地服务:AllenService。  

由前面的学习知道，要完成一个自己的本地服务，需要有IAllenService接口、BnAllenService服务Stub、AllenService、BpAllenService。UML图如下<!--more-->：  

![AllenService](http://7xn1yt.com1.z0.glb.clouddn.com/AllenService.png)

各文件的路径如下：  

+ frameworks/base/include/allen/IAllenService.h,AllenService.h

+ frameworks/base/libs/allen/IAllenService.cpp,AllenService.cpp

+ frameworks/base/cmds/allen/main_allenclient.cpp,main_allenservice.cpp

1.IAllenService.h文件  

该头文件中包含了接口描述符以及枚举类。另外，与Android中的服务接口文件类似，这里将BnAllenService的定义也放在该头文件中，所以该头文件的代码如下：  

    #ifndef FRAMEWORKS_BASE_INCLUDE_IALLENSERVICE_H_
    #define FRAMEWORKS_BASE_INCLUDE_IALLEN_SERVICE_H_

    #include <utils/Errors.h>
    #include <utils/KeyedVector.h>
    #include <utils/RefBase.h>
    #include <utils/String8.h>
    #include <binder/IInterface.h>
    #include <binder/Parcel.h>

    #include <AllenService.h>

    namespace android{

        #define MATH_ERROR int32(-1)
        #define ALLEN_DESCRIPTOR "android.allen.IAllenService"

        enum{
            ALLEN_PRINT=IBinder::FIRST_CALL_TRANSACTION,
            ALLEN_ADD,
            ALLEN_SUB
        };

        class IAllenService:public IInterface
        {
            public:
                //宏定义
                DECLARE_META_INTERFACE(AllenService);

                virtual status_t print(const char*str)=0;
                virtual int32 add(int32 a,int32 b)=0;
                virtual int32 sub(int32 a,int32 b)=0;
        };

        class BnAllenService:public BnInterface<IAllenService>
        {
            public:
                virtual status_t onTransact(uint32_t code,
                                            const Parcel& data,
                                            Parcel*reply,
                                            uint32_t flags=0);
        };
    };//namespace android

    #endif


2. AllenService.h  

AllenService头文件的代码如下：  

    #ifndef FRAMEWORKS_BASE_INCLUDE_ALLEN_ALLENSERVICE_H_
    #define FRAMEWORKS_BASE_INCLUDE_ALLEN_ALLENSERVICE_H_

    #include <utils/Log.h>
    #include <utils/threads.h>
    #include <utils/List.h>
    #include <utils/Errors.h>
    #include <utils/KeyedVector.h>
    #include <utils/String8.h>
    #include <utils/Vector.h>

    #include <IAllenService.h>

    namespace android{

        class AllenService:public BnAllenService{
            public:
                static void instantiate();
                virtual status_t print(const char*str);
                virtual int32 add(int32 a,int32 b);
                virtual int32 sub(int32 a,int32 b);

                virtual status_t onTransact(
                        uint32_t code,
                        const Parcel& data,
                        Parcel* reply,
                        uint32_t flags);
            private:
                AllenService();
                virtual ~AllenService();
        };
    }; //namespace android

    #endif

3. IAllenService.cpp  

该文件中不仅有IAllenService中宏接口的实现，还有BnAllenService的实现，另外，还有BpAllenService的定义以及实现。  

    #include <stdint.h>
    #include <sys/types.h>

    #include <binder/Parcel.h>
    #include <binder/IMemory.h>
    #include <allen/IAllenService.h>
    #include <allen/AllenService.h>

    #include <utils/Errors.h>

    namespace android{

        class BpAllenService: public BpInterface<IAllenService>
        {
            public:
                BpAllenService(const sp<IBinder>& impl):BpInterface<IAllenService>(impl)
                {

                }

                virtual status_t print(const char*str)
                {
                    Parcel data,reply;
                    data.writeInterfaceToken(IAllenService::getInterfaceDescriptor());
                    data.writeCString(str);

                    status_t status=remote()->transact(ALLEN_PRINT,data,&reply);
                    if(status!=NO_ERROR){
                        LOGE("print str error:%s",strerror(-status));
                    }else{
                        status=reply.readInt32();
                    }

                    return status;
                }

                virtual int32 add(int32 a,int32 b)
                {
                    Parcel data,reply;
                    data.writeInterfaceToken(IAllenService::getInterfaceDescriptor());
                    data.writeInt32(a);
                    data.writeInt32(b);

                    status_t status=remote()->transact(ALLEN_ADD,data,&reply);

                    return reply.readInt32();
                }

                virtual int32 sub(int32 a,int32 b)
                {
                    Parcel data,reply;
                    data.writeInterfaceToken(IAllenService::getInterfaceDescriptor());
                    data.writeInt32(a);
                    data.writeInt32(b);

                    status_t status=remote()->transact(ALLEN_SUB,data,&reply);

                    return reply.readInt32();
                }
        };

        //宏实现
        IMPLEMENT_META_INTERFACE(AllenService,ALLEN_DESCRIPTOR)

        status BnAllenService::onTransact(uint32_t code,const Parcel& data,Parcel* reply,uint32_t flags)
        {
            switch(code){
                case ALLEN_PRINT:{
                    CHECK_INTERFACE(IAllenService,data,reply);

                    const char*str=data.readCString();

                    reply->writeInt32(print(str));
                    return NO_ERROR;
                }break;
                case ALLEN_ADD:{
                    CHECK_INTERFAC(IAllenService,data,reply);

                    int32 a=data.readInt32();
                    int32 b=data.readInt32();

                    reply->writeInt32(this->add(a,b));

                    return NO_ERROR;
                }break;
                case ALLEN_SUB:{
                    CHECK_INTERFACE(IAllenService,data,reply);

                    int32 a=data.readInt32();
                    int32 b=data.readInt32();

                    reply->writeInt32(this->sub(a,b));
                    return NO_ERROR;
                }break;
                default:
                    return BBinder::onTransact(code,data,reply,flags);

            }
        }

    };

4.AllenService.cpp  

该文件实现服务接口。代码如下：  

    namespace android{

        void AllenService::instantiate(){
            defaultServiceManager()->addService(
                String16(ALLEN_DESCRIPTOR),new AllenService());
        }

        status_t AllenService::print(const char*str){
            LOGI("%s\n",str);
            printf("%s\n",str);
            return NO_ERROR;
        }

        int32 AllenService::add(int32 a,int32 b){
            LOGI("a=%d,b=%d",a,b);

            return a+b;
        }

        int32 AllenService::sub(int32 a,int32 b){
            LOGI("a=%d,b=%d",a,b);

            return a-b;
        }
    };

 5.main_allenservice.cpp  

创建好AllenService后，如果想要运行它，就需要有服务客户端和Service Server进程，服务代理运行在服务客户端中，服务运行在Service Server进程中。如下图所示：  

![client_vs_service](http://7xn1yt.com1.z0.glb.clouddn.com/client_vs_service.png)

main_allenservice.cpp的代码如下：  

    int main(int argc,char** argv)
    {
        sp<ProcessState>proc(ProcessState::self());
        sp<IServiceManager>sm=defaultServiceManager();
        LOGI("ServiceManager: %p",sm.get());

        AllenService::instantiate();

        ProcessState::self()->startThreadPool();
        ProcessState::self()->joinThreadPool();

        return 0;
    }

6.main_allenclient.cpp  

main_allenclient.cpp的代码如下：  

    int main(int argc,char *argv[])
    {
        LOGI("allenservice cleint is now starting");

        sp<IServiceManager>sm=defaultServiceManager();
        sp<IBinder>binder;
        sp<IAllenService>allenService;

        do{
            binder=sm->getService(String16(ALLEN_DESCRIPTOR));
            if(binder!=0)
            {
                break;
            }
            LOGI("AllenService is not working,try again...");
            usleep(500000);
        }while(true);

        allenService=interface_cast<IAllenService>(binder);
        allenService->print("Hello world!\nI'm Allen and I'm coning.\n");
        allenService->add(3,6);
        allenService->sub(1,1);

        return 0;
    }

7.编译AllenService系统服务  

若想编译AllenService服务代码，需要将AllenService的源码包含进Android平台源码中，再进行整体编译。首先保证头文件在/frameworks/base/include/allen下，源代码文件在/frameworks/base/libs/allen下，而后编写Android.mk文件，编写完成后，将其放到源代码目录中，以便将源代码与Android平台代码一起编译。  
Android.mk文件内容如下：  

    LOCAL_PATH:=$(call my-dir)
    include &(CLEAR_VARS)

    LOCAL_SRC_FILES:= \
        IAllenService.cpp \
        AllenService.cpp

    LOCAL_SHARED_LIBRARES :=\
        libcutils \
        libutils \
        libbinder \

    LOCAL_PRELINK_MODULE := false

    LOCAL_MODULE :=liballen

    include $(BUILD_SHARED_LIBRARY)

接着，编写运行main_allenservice.cpp和main_allenclient.cpp的Android.mk文件，内容如下：  

    LOCAL_PATH:= $(call my-dir)
    include $(CLEAR_VARS)

    LOCAL_SRC_FILES:= \
        main_allenserver.cpp

    LOCAL_SHARED_LIBRARIES := \
        libutils \
        libbinder \
        liballen
        base := $(LOCAL_APTH)/../..

    LOCAL_MODULE:= allenservice

    incldue $(BUILD_EXECUTABLE)

    include $(CLEAR_VARS)

    LOCAL_SRC_FILES:= \
        main_allenclient.cpp

    LOCAL_SHARED_LIBRARIES := \
        libutils \
        libbinder \
        liballen

    base :=$(LOCAL_PATH)/../..
        LOCAL_MODULE:=allenclient

    include $(BUILD_EXECUTABLE)




