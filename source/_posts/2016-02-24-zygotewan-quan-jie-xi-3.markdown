---
layout: post
title: "Zygote完全解析(3)"
date: 2016-02-24 09:06:53 +0800
comments: true
categories: android_deep_analysis
---

本文是Zygote完全解析系统的第3篇也是最后一篇Blog，主要是分析SystemServer运行后是如何处理来自所绑定的socket的请求，从而运行新的应用程序的。  

通过[Zygote完全解析(1)](http://blog.imallen.wang/blog/2016/02/23/zygotewan-quan-jie-xi-1/)中的ZygoteInit.main()方法中可知，在运行SystemServer后，如果不是ZYGOTE_FORK_MODE，程序会进入一个循环，处理来自所绑定的socket的请求，对应的代码 runSelectLoopMode()如下<!--more-->：  

    /**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     *
     * @throws MethodAndArgsCaller in a child process when a main() should
     * be executed.
     */
    private static void runSelectLoopMode() throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList();
        ArrayList<ZygoteConnection> peers = new ArrayList();
        FileDescriptor[] fdArray = new FileDescriptor[4];

        //code_1
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        int loopCount = GC_LOOP_COUNT;
        while (true) {
            int index;

            /*
             * Call gc() before we block in select().
             * It's work that has to be done anyway, and it's better
             * to avoid making every child do it.  It will also
             * madvise() any free memory as a side-effect.
             *
             * Don't call it every time, because walking the entire
             * heap is a lot of overhead to free a few hundred bytes.
             */
            if (loopCount <= 0) {
                gc();
                loopCount = GC_LOOP_COUNT;
            } else {
                loopCount--;
            }


            try {
                fdArray = fds.toArray(fdArray);
                //code_2
                index = selectReadable(fdArray);
            } catch (IOException ex) {
                throw new RuntimeException("Error in select()", ex);
            }

            if (index < 0) {
                throw new RuntimeException("Error in select()");
            } else if (index == 0) {
                //code_3
                ZygoteConnection newPeer = acceptCommandPeer();
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                boolean done;
                //code_4
                done = peers.get(index).runOnce();

                if (done) {
                    peers.remove(index);
                    fds.remove(index);
                }
            }
        }
    }

1)在ZygoteInit.main()方法的前半部分中，将套接字与dev/socket/zygote绑定在一起。本行首先将套接字的描述符添加到描述符数组中，保存在数组的第0个Index中。程序将使用该描述符处理来自外部的连接请求;  
2)selectReadable()是一个被注册为JNI本地方法的本地函数。该方法用来监视参数传递过来的文件描述符数组，若描述符目录中存在相关事件，则返回其在数组中的索引;  
3)该部分代码用来处理Index为0的socket描述符中发生的输入输出事件。为了处理传递给dev/socket/zygote的socket的连接请求，程序先创建出ZygoteConnection类的对象。在ZygoteConnection类的构造方法中创建输入输出流，而后生成Credentials,检查请求连接一方的访问权限。为了处理ZygoteConnection对象的输入输出事件，将socket描述符添加到socket描述符数组fds中。被添加的socket描述符的输入输出事件在下一个循环中由selectReadable()方法进行检测;  
4)此行代码用于处理新连接的输入输出socket，并生成新的Android应用程序，下面是runOnce()的代码：  

     /**
     * Reads one start command from the command socket. If successful,
     * a child is forked and a {@link ZygoteInit.MethodAndArgsCaller}
     * exception is thrown in that child while in the parent process,
     * the method returns normally. On failure, the child is not
     * spawned and messages are printed to the log and stderr. Returns
     * a boolean status value indicating whether an end-of-file on the command
     * socket has been encountered.
     *
     * @return false if command socket should continue to be read from, or
     * true if an end-of-file has been encountered.
     * @throws ZygoteInit.MethodAndArgsCaller trampoline to invoke main()
     * method in child process
     */
    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            //code_1
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            Log.w(TAG, "IOException on command socket " + ex.getMessage());
            closeSocket();
            return true;
        }

        if (args == null) {
            // EOF reached.
            closeSocket();
            return true;
        }

        /** the stderr of the most recent request, if avail */
        PrintStream newStderr = null;

        if (descriptors != null && descriptors.length >= 3) {
            newStderr = new PrintStream(
                    new FileOutputStream(descriptors[2]));
        }

        int pid;

        try {
            //code_2
            parsedArgs = new Arguments(args);

            applyUidSecurityPolicy(parsedArgs, peer);
            applyDebuggerSecurityPolicy(parsedArgs);
            applyRlimitSecurityPolicy(parsedArgs, peer);
            applyCapabilitiesSecurityPolicy(parsedArgs, peer);

            int[][] rlimits = null;

            if (parsedArgs.rlimits != null) {
                rlimits = parsedArgs.rlimits.toArray(intArray2d);
            }
            //code_3
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids, parsedArgs.debugFlags, rlimits);
        } catch (IllegalArgumentException ex) {
            logAndPrintError (newStderr, "Invalid zygote arguments", ex);
            pid = -1;
        } catch (ZygoteSecurityException ex) {
            logAndPrintError(newStderr,
                    "Zygote security policy prevents request: ", ex);
            pid = -1;
        }

        if (pid == 0) {
            // in child
            //code_4
            handleChildProc(parsedArgs, descriptors, newStderr);
            // should never happen
            return true;
        } else { /* pid != 0 */
            // in parent...pid of < 0 means failure
            //code_5
            return handleParentProc(pid, descriptors, parsedArgs);
        }
    }

1)读取请求信息，请求信息包含创建新进程的参数选项，以"line count+字符串数组"形式存在。  
2)分析请求信息中的字符串数组，为运行进程设置好各个选项，具体包括设置应用程序的gid、uid,调试标记处理，设置rlimit,以及检查运行权限等;  
3)创建新进程，Zygote.forkAndSpecialize()方法接收上面分析好的参数，调用Zygote类的本地方法forkAndSpecialize();  
4)创建好进程之后，会返回一个pid值，根据pid是否为0就知道当前的程序是运行在子进程中还是父进程中(因为子进程的pid实际上会传给父进程，而在子进程中pid还是初始值)，如果是子进程，则调用handleChildProc()方法，如果是在父进程中则调用handleParentProc()方法.而handleChildProc()方法在上一篇博客中曾经讲过，它的主要作用就是加载新进程所需要的类，并调用类的main()方法，启动新的应用程序，直到运行的Android应用程序终止，该函数才会返回。  
5)handleParentProc()中根据pid的返回值进行判断，如果pid>0表示子进程创建成功，则通过ZygoteInit.sepgid()方法将子进程放入与它相同的进程组中；handleParentProc()的主要作用是请求完成后，断开连接，关闭socket.看下面的代码很容易就能理解：  

    /**
     * Handles post-fork cleanup of parent proc
     *
     * @param pid != 0; pid of child if &gt; 0 or indication of failed fork
     * if &lt; 0;
     * @param descriptors null-ok; file descriptors for child's new stdio if
     * specified.
     * @param parsedArgs non-null; zygote args
     * @return true for "exit command loop" and false for "continue command
     * loop"
     */
    private boolean handleParentProc(int pid,
            FileDescriptor[] descriptors, Arguments parsedArgs) {

        if(pid > 0) {
            // Try to move the new child into the peer's process group.
            try {
                ZygoteInit.setpgid(pid, ZygoteInit.getpgid(peer.getPid()));
            } catch (IOException ex) {
                // This exception is expected in the case where
                // the peer is not in our session
                // TODO get rid of this log message in the case where
                // getsid(0) != getsid(peer.getPid())
                Log.i(TAG, "Zygote: setpgid failed. This is "
                    + "normal if peer is not in our session");
            }
        }

        try {
            if (descriptors != null) {
                for (FileDescriptor fd: descriptors) {
                    ZygoteInit.closeDescriptor(fd);
                }
            }
        } catch (IOException ex) {
            Log.e(TAG, "Error closing passed descriptors in "
                    + "parent process", ex);
        }

        try {
            mSocketOutStream.writeInt(pid);
        } catch (IOException ex) {
            Log.e(TAG, "Error reading from command socket", ex);
            return true;
        }

        /*
         * If the peer wants to use the socket to wait on the
         * newly spawned process, then we're all done.
         */
        if (parsedArgs.peerWait) {
            try {
                mSocket.close();
            } catch (IOException ex) {
                Log.e(TAG, "Zygote: error closing sockets", ex);
            }
            return true;
        }
        return false;
    }

至此，我们就分析完了ZygoteInit类加载新应用程序类并调用main()方法执行的整个过程，类似于Linux本地环境下调用dlopen()方法使用动态库。新运行的应用程序由ZygoteInit类动态加载，共用装载到父进程生成的虚拟机中的代码。并且，共用应用程序Framework中的类以及资源的链接信息，这大大加快了应用程序创建与启动的速度。
