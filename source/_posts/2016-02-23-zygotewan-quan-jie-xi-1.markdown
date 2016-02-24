---
layout: post
title: "Zygote完全解析(1)"
date: 2016-02-23 20:31:03 +0800
comments: true
categories: android_deep_analysis
---
1.什么是Zygote  

Zygote的字面意思是"受精卵",我们都知道，有丝分裂的个体最初都是从受精卵发育而来，意思非常明显，在Android系统中也一样，运行新的应用，需要跟Zygote进程结合后才能执行。  

我们知道，Android的应用程序是由Java编写的，它们不能直接以本地进程的形态运行在Linux上，只能运行在Dalvik虚拟机中(当然，在5.0之后采用ART了，但是不影响此处的理解)。每个应用程序都运行在各自的虚拟机中，应用程序每次运行都要重新初始化并启动虚拟机，这个过程会耗费相当长时间，是导致应用程序启动慢的原因之一。  

为了很好地解决这个问题，在Android中，应用程序运行前，Zygote进程通过共享已运行的虚拟机的代码与内存信息，缩短应用程序运行所耗费的时间。并且，它会预先将应用程序要使用的Android Framework中的类与资源加载到内存中，并组织形成所用资源的链接信息。因此，新运行的Android应用程序在使用所需资源时不必每次重新形成资源的链接信息，从而提高运行速度<!--more-->。  

2.Zygote孵化进程分析  

2.1 Zygote进程的产生  

Android系统启动后运行的首个进程就是init进程。init进程启动完系统运行所需要的各种Daemon后，就启动Zygote进程，这是系统的第一个Zygote进程。该过程如下图所示：  

![init](http://7xn1yt.com1.z0.glb.clouddn.com/init.png)

利用adb命令可以看到手机上运行的进程间的派生关系，如下图所示：  

![zygote](http://7xn1yt.com1.z0.glb.clouddn.com/zygote.png)

从图片的最顶部可以看到zygote的PPID(Parent Process ID,父进程)为1，即init进程，而zygote进程ID为263(注意：这个ID在不同的手机上不一样，因为启动的daemons数量不同)，再去查找与263有关的，可以发现launcher,PushService,systemui,location等各种进程的父进程都是zygote，这进一步验证了上面的观点。  

2.2 Zygote孵化进程的原理  

为了通过比较进程的生成过程，先来看一下在Linux系统中是如何创建一个新进程的。先看下图：  

![linux_fork](http://7xn1yt.com1.z0.glb.clouddn.com/linux_fork.png)

如上图所示，父进程A调用fork()函数创建子进程A1,A1共享A中的内存结构信息与库连接信息，直到子进程A1调用exec('B'),将新进程B的代码装载到内存中。此时，父进程A的内存信息被清除，并重新分配内存，以便运行被装载的B进程，接着形成新的库连接信息，以供进程B使用。  

那么，在Android中，一个新的应用进程是如何创建并运行的呢？Google在发布Android平台讲述平台特性时曾讲到, Zygote是Android系统的一个主要特征，它通过COW(Copy On Write,我个人将其翻译为写入时才复制)方式对运行在内存中的进程实现了最大程度的复用，并通过库共享有效地降低了内存的使用量.  

如下图所示，Zygote进程调用fork()函数创建出子进程Zygote1,它共享父进程Zygote的代码区与连接信息。但是,新的Android应用程序A并非通过fork()来重新装载已有进程的代码区，而是被动态地加载到复制出的Dalvik虚拟机上。而后，Zygote进程将执行流程交给应用程序A的方法，应用程序A开始运行。  

![zygote_fork](http://7xn1yt.com1.z0.glb.clouddn.com/zygote_fork2.png)

如下图所示，由于Zygote启动后，初始化并运行Dalvik虚拟机，而后将需要的类与资源加载到内存中。随后调用fork()创建出Zygote1子进程，接着子进程Zygote1动态加载并运行Andriod应用程序A.运行的应用程序A会使用Zygote已经初始化并启动运行的Dalvik虚拟机代码，通过使用已加载至内存中的类与资源来加快运行速度。  

![zygote_load](http://7xn1yt.com1.z0.glb.clouddn.com/zygote_load2.png)

2.3 由app_process运行ZygoteInit class  

与其他本地服务或Daemon不同的是，Zygote由Java编写而成，不能直接由init进程启动运行。若想运行Zygote类，必须先生成Dalvik虚拟机，瑞在Dalvik虚拟机上装载运行ZygoteInit类，而执行这一任务的就是app_process进程。下面来分析相应的源码(在frameworks/base/cmds/app_process/app_main.cpp中)：  

	int main(int argc, const char* const argv[])
	{
	    // These are global variables in ProcessState.cpp
	    mArgC = argc;
	    mArgV = argv;
	    
	    mArgLen = 0;
	    for (int i=0; i<argc; i++) {
	        mArgLen += strlen(argv[i]) + 1;
	    }
	    mArgLen--;

	    AppRuntime runtime;
	    const char *arg;
	    const char *argv0;

	    argv0 = argv[0];

	    // Process command line arguments
	    // ignore argv[0]
	    argc--;
	    argv++;

	    // Everything up to '--' or first non '-' arg goes to the vm
	    
	    int i = runtime.addVmArguments(argc, argv);

	    // Next arg is parent directory
	    if (i < argc) {
	        runtime.mParentDir = argv[i++];
	    }

	    // Next arg is startup classname or "--zygote"
	    if (i < argc) {
	        arg = argv[i++];
	        if (0 == strcmp("--zygote", arg)) {
	            bool startSystemServer = (i < argc) ? 
	                    strcmp(argv[i], "--start-system-server") == 0 : false;
	            setArgv0(argv0, "zygote");
	            set_process_name("zygote");
	            runtime.start("com.android.internal.os.ZygoteInit",
	                startSystemServer);
	        } else {
	            set_process_name(argv0);

	            runtime.mClassName = arg;

	            // Remainder of args get passed to startup class main()
	            runtime.mArgC = argc-i;
	            runtime.mArgV = argv+i;

	            LOGV("App process is starting with pid=%d, class=%s.\n",
	                 getpid(), runtime.getClassName());
	            runtime.start();
	        }
	    } else {
	        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
	        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
	        app_usage();
	        return 10;
	    }

	}

显然，在main()函数中创建了AppRuntime对象。其中AppRuntime类继承自AndroidRuntime类，而AndroidRuntime类的作用是初始化并运行Dalvik虚拟机，为运行Android应用程序做好准备。  

int i=runtime.addVmArguments(argc,argv);的作用是在运行Dalvik虚拟机之前，通过AppRuntime对象，分析环境变量以及运行的参数，并以此生成虚拟机选项。  

分析init.rc文件可发现init进程在运行app_process时根据如下规则传递参数：  

	app_process [java-options] cmd-dir start-class-name [options]

java-options:传递给虚拟机的选项，必须以"-"开始;  
cmd-dir:所要运行的进程所在的目录  
start-class-name:要在虚拟机中运行的类的名称。app_process会将指定的类加载到虚拟机中，而后调用类的main()方法。  
options:要传递给类的选项  

app_process服务运行时，init.rc文件中的运行命令如下：  

	/system/bin/app_process -Xzygote /system/bin --zygote --start-system-server

根据参数规则，可以知道:  
1)**-Xzygote是指要传递给VM的选项，-Xzygote选项用于区分要在虚拟机中运行的类是Zygote，还是在Zygote中运行的其他Android应用程序**，它会被保存在AppRuntime的mOption变量中，如代码int i=runtime.addVmArguments(argc,argv);所示。  

2)在代码 runtime.mParentDir = argv[i++]; 中，运行目录参数"syste/bin"被保存到AppRuntime的mParentDir变量中。  

3)第三个参数用来指定加载到虚拟机中的类的名称,"--zygote"表示加载com.android.internal.os.ZygoteInit类;  

4)最后一个参数"--start-system-server"作为选项传递给生成的类，用于启动系统服务器。  

之后调用runtime.start()方法将指定的类加载至虚拟机中，由于代码很简单，不再赘述。  

2.4 AndroidRuntime创建虚拟机  

下面是AndroidRuntime中创建虚拟机的主要代码：  

	void AndroidRuntime::start(const char*className, const bool startSystemServer)
	{
		...
		if(JNI_CreateJavaVM(&mJavaVM,&env,&initArgs)<0){
			LOGE("JNI_CreateJavaVM failed\n");
			goto bail;
		}
		...
	}

显然，start()函数通过调用JNI_CreateJavaVM()函数来创建并运行虚拟机，其函数原型如下：  

	jint JNI_CreateJavaVM(JavaVM**p_vm, JNIEnv**p_env, void*vm_args)

JavaVM**p_vm:生成的JavaVM类的接口指针  
JNIEnv**p_env:JNIEnv类的接口指针，方便访问虚拟机  
void*vm_args:已设置的虚拟机选项  

至于JNI相关的知识，在上一篇博客中已经介绍过了，此处不再进行赘述。  
虚拟机创建后，接下来，注册要在虚拟机中使用的JNI函数，调用startReg()方法实现。需要注册的方法数组如下所示：  

	static const RegJNIRec gRegJNI[] = {
	    REG_JNI(register_android_debug_JNITest),
	    REG_JNI(register_com_android_internal_os_RuntimeInit),
	    REG_JNI(register_android_os_SystemClock),
	    REG_JNI(register_android_util_EventLog),
	    REG_JNI(register_android_util_Log),
	    REG_JNI(register_android_util_FloatMath),
	    REG_JNI(register_android_text_format_Time),
	    REG_JNI(register_android_pim_EventRecurrence),
	    REG_JNI(register_android_content_AssetManager),
	    REG_JNI(register_android_content_StringBlock),
	    REG_JNI(register_android_content_XmlBlock),
	    REG_JNI(register_android_emoji_EmojiFactory),
	    REG_JNI(register_android_security_Md5MessageDigest),
	    REG_JNI(register_android_text_AndroidCharacter),
	    REG_JNI(register_android_text_AndroidBidi),
	    REG_JNI(register_android_text_KeyCharacterMap),
	    REG_JNI(register_android_os_Process),
	    REG_JNI(register_android_os_Binder),
	    REG_JNI(register_android_view_Display),
	    REG_JNI(register_android_nio_utils),
	    REG_JNI(register_android_graphics_PixelFormat),
	    REG_JNI(register_android_graphics_Graphics),
	    REG_JNI(register_android_view_Surface),
	    REG_JNI(register_android_view_ViewRoot),
	    REG_JNI(register_com_google_android_gles_jni_EGLImpl),
	    REG_JNI(register_com_google_android_gles_jni_GLImpl),
	    REG_JNI(register_android_opengl_jni_GLES10),
	    REG_JNI(register_android_opengl_jni_GLES10Ext),
	    REG_JNI(register_android_opengl_jni_GLES11),
	    REG_JNI(register_android_opengl_jni_GLES11Ext),
	    REG_JNI(register_android_opengl_jni_GLES20),

	    REG_JNI(register_android_graphics_Bitmap),
	    REG_JNI(register_android_graphics_BitmapFactory),
	    REG_JNI(register_android_graphics_Camera),
	    REG_JNI(register_android_graphics_Canvas),
	    REG_JNI(register_android_graphics_ColorFilter),
	    REG_JNI(register_android_graphics_DrawFilter),
	    REG_JNI(register_android_graphics_Interpolator),
	    REG_JNI(register_android_graphics_LayerRasterizer),
	    REG_JNI(register_android_graphics_MaskFilter),
	    REG_JNI(register_android_graphics_Matrix),
	    REG_JNI(register_android_graphics_Movie),
	    REG_JNI(register_android_graphics_NinePatch),
	    REG_JNI(register_android_graphics_Paint),
	    REG_JNI(register_android_graphics_Path),
	    REG_JNI(register_android_graphics_PathMeasure),
	    REG_JNI(register_android_graphics_PathEffect),
	    REG_JNI(register_android_graphics_Picture),
	    REG_JNI(register_android_graphics_PorterDuff),
	    REG_JNI(register_android_graphics_Rasterizer),
	    REG_JNI(register_android_graphics_Region),
	    REG_JNI(register_android_graphics_Shader),
	    REG_JNI(register_android_graphics_Typeface),
	    REG_JNI(register_android_graphics_Xfermode),
	    REG_JNI(register_android_graphics_YuvImage),
	    REG_JNI(register_com_android_internal_graphics_NativeUtils),

	    REG_JNI(register_android_database_CursorWindow),
	    REG_JNI(register_android_database_SQLiteCompiledSql),
	    REG_JNI(register_android_database_SQLiteDatabase),
	    REG_JNI(register_android_database_SQLiteDebug),
	    REG_JNI(register_android_database_SQLiteProgram),
	    REG_JNI(register_android_database_SQLiteQuery),
	    REG_JNI(register_android_database_SQLiteStatement),
	    REG_JNI(register_android_os_Debug),
	    REG_JNI(register_android_os_FileObserver),
	    REG_JNI(register_android_os_FileUtils),
	    REG_JNI(register_android_os_ParcelFileDescriptor),
	    REG_JNI(register_android_os_Power),
	    REG_JNI(register_android_os_StatFs),
	    REG_JNI(register_android_os_SystemProperties),
	    REG_JNI(register_android_os_UEventObserver),
	    REG_JNI(register_android_net_LocalSocketImpl),
	    REG_JNI(register_android_net_NetworkUtils),
	    REG_JNI(register_android_net_TrafficStats),
	    REG_JNI(register_android_net_wifi_WifiManager),
	    REG_JNI(register_android_os_MemoryFile),
	    REG_JNI(register_com_android_internal_os_ZygoteInit),
	    REG_JNI(register_android_hardware_Camera),
	    REG_JNI(register_android_hardware_SensorManager),
	    REG_JNI(register_android_media_AudioRecord),
	    REG_JNI(register_android_media_AudioSystem),
	    REG_JNI(register_android_media_AudioTrack),
	    REG_JNI(register_android_media_JetPlayer),
	    REG_JNI(register_android_media_ToneGenerator),

	    REG_JNI(register_android_opengl_classes),
	    REG_JNI(register_android_bluetooth_HeadsetBase),
	    REG_JNI(register_android_bluetooth_BluetoothAudioGateway),
	    REG_JNI(register_android_bluetooth_BluetoothSocket),
	    REG_JNI(register_android_bluetooth_ScoSocket),
	    REG_JNI(register_android_server_BluetoothService),
	    REG_JNI(register_android_server_BluetoothEventLoop),
	    REG_JNI(register_android_server_BluetoothA2dpService),
	    REG_JNI(register_android_server_Watchdog),
	    REG_JNI(register_android_message_digest_sha1),
	    REG_JNI(register_android_ddm_DdmHandleNativeHeap),
	    REG_JNI(register_android_location_GpsLocationProvider),
	    REG_JNI(register_android_backup_BackupDataInput),
	    REG_JNI(register_android_backup_BackupDataOutput),
	    REG_JNI(register_android_backup_FileBackupHelperBase),
	    REG_JNI(register_android_backup_BackupHelperDispatcher),
	};

2.5 运行ZygoteInit类  

在AndroidRuntime的start()函数中，当创建虚拟机并注册相应的本地方法之后，接下来还会运行相应的Java类的main()方法。所谓的相应的Java类是指ZygoteInit类或者与Application相关的ActivityThread类等。下面是主要代码：  

	    /*
	     * Start VM.  This thread becomes the main thread of the VM, and will
	     * not return until the VM exits.
	     */
	    jclass startClass;
	    jmethodID startMeth;

	    slashClassName = strdup(className);
	    for (cp = slashClassName; *cp != '\0'; cp++)
	        if (*cp == '.')
	            *cp = '/';

	    startClass = env->FindClass(slashClassName);
	    if (startClass == NULL) {
	        LOGE("JavaVM unable to locate class '%s'\n", slashClassName);
	        /* keep going */
	    } else {
	        startMeth = env->GetStaticMethodID(startClass, "main",
	            "([Ljava/lang/String;)V");
	        if (startMeth == NULL) {
	            LOGE("JavaVM unable to find main() in '%s'\n", className);
	            /* keep going */
	        } else {
	            env->CallStaticVoidMethod(startClass, startMeth, strArray);

	#if 0
	            if (env->ExceptionCheck())
	                threadExitUncaughtException(env);
	#endif
	        }
	    }

如果认真看了我的博客--[JNI完全解析](http://blog.imallen.wang/blog/2016/02/19/jniwan-quan-jie-xi/)的话，应该很容易理解这里的env->GetStaticMethodId(startClass,"main","([Ljava/lang/String:)V")表示获得main方法的ID，然后在env->CallStaticVoidMethod(startClass,startMeth,strArray); 中调用main()方法。  

显然，要了解ZygoteInit做了些什么事情，只需要看一下ZygoteInit的main()方法的代码：  

	    public static void main(String argv[]) {
        try {
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();
            
            //绑定socket，接收新Android应用程序运行请求
            registerZygoteSocket();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            //加载Android Application Framework使用的类与资源
            preloadClasses();
            //cacheRegisterMaps();
            preloadResources();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());

            if (SamplingProfilerIntegration.isEnabled()) {
                SamplingProfiler sp = SamplingProfiler.getInstance();
                sp.pause();
                SamplingProfilerIntegration.writeZygoteSnapshot();
                sp.shutDown();
            }

            // Do an initial gc to clean up after startup
            gc();

            // If requested, start system server directly from Zygote
            if (argv.length != 2) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }

            //运行SystemServer
            if (argv[1].equals("true")) {
                startSystemServer();
            } else if (!argv[1].equals("false")) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }

            Log.i(TAG, "Accepting command socket connections");

            if (ZYGOTE_FORK_MODE) {
                runForkMode();
            } else {
            	//处理新Android应用程序请求
                runSelectLoopMode();
            }

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }

下一节我们就根据该源码来分析ZygoteInit做了哪些事情。  

2.6 ZygoteInit类的功能  

在上一小节的最后我们给出了ZygoteInit的main()函数代码，并且给出了相应的注释。概括起来，主要完成了以下4个工作:  
1)registerZygoteSocket(); 语句用来绑定socket，以便接收新的Android应用程序的运行请求。为了从ActivityManager接收新的Android应用程序的运行请求，Zygote使用UDS(Unix Domain Socket)，init进程在运行app_process时，使用init.rc文件中以"/dev/zygote"形式注册的socket;  
2)该行用于将应用程序框架中的类、平台资源(图像、XML信息、字符串等)预先加载到内存中。新进程直接使用这些类与资源，而不需要重新加载它们，这大大加快了程序的执行速度；  
3)通过app_process运行zygote时，参数"--start-system-server"会调用startSystemServer()方法启动系统服务器，系统服务器用来运行Android平台需要的一些主要的本地服务(主要是SurfaceFlinger,AudioFlinger,MediaPlayerService,CameraService等);   
4)runSelectLoopMode();的作用是监视UDS,若收到新的Android应用程序生成请求，则进入处理循环.  

用一张图来表示ZygoteInit的main()方法完成的功能就是：  

![zygote_main](http://7xn1yt.com1.z0.glb.clouddn.com/zygote_main.png)

关于以上4个工作具体是如何实现的，将在下一篇博客中详细解读，敬请关注。 