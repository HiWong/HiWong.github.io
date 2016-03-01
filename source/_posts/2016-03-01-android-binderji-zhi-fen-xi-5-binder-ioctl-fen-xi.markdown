---
layout: post
title: "Android Binder机制分析(5) binder_ioctl()分析"
date: 2016-03-01 10:11:50 +0800
comments: true
categories: android_deep_analysis
---

**引言**  

在博客[Android Binder机制(3)本地服务注册过程](http://blog.imallen.wang/blog/2016/02/27/android-binderji-zhi-3-ben-di-fu-wu-zhu-ce-guo-cheng/)这篇博客中我们详细讲解了本地服务的注册过程，除了一个地方之外，那就是IPCThreadState::waitForResponse()方法中的talkWithDriver()，而在talkWithDriver()中调用了binder_ioctl()，由于内容太多，所以专门写一篇博客进行分析。  

实际上，不只是在服务注册过程中会调用到Binder Driver中的binder_ioctl()，在服务检索、服务使用阶段都会调用到binder_ioctl()，所以这篇博客将分这3个阶段详细讲解binder_ioctl()方法。  

并且，这3个阶段均可按Binder数据的传递划分为6个阶段，如下所示<!--more-->：  

![binder_flow](http://7xn1yt.com1.z0.glb.clouddn.com/binder_flow.png)

本文中对于服务注册、服务检索、服务使用都分这6个阶段进行讲解。  

另外，由于在博客[Android Binder机制(2) ContextManager注册过程分析](http://blog.imallen.wang/blog/2016/02/27/android-binderji-zhi-2-contextmanagerzhu-ce-guo-cheng-fen-xi/)中详细分析过binder_open和binder_mmap()函数，这里就不再讲解了，不了解的小伙伴可以移步到那篇Blog中。  

在开始之前，我们先补充一些基础知识。  

0.1 ioctl的BINDER_WRITE_READ命令用来请求Binder Driver发送或接收IPC数据以及IPC应答数据  

0.2 binder_ioctl()函数负责在两个进程间收发IPC数据以及IPC应答数据。因此，在分析binder_ioctl()函数时，必然要涉及到发送IPC数据和接收IPC数据的两个进程。  

0.3 binder_write_read结构体的定义如下：  

     struct binder_write_read{
        //bytes to write
        signed long write_size; 
        //bytes consumed by driver
        unsigned long write_buffer;
        //bytes to read
        signed long read_size;
        //bytes consumed by driver
        signed long read_consumed;
        unsigned long read_buffer;
    }

+ write_buffer成员变量用于发送IPC数据或IPC应答数据。而read_buffer成员变量用来接收来自Binder Driver的数据，即Binder Driver在接收IPC数据或IPC应答数据后，将它们保存到read_buffer中，而后再传递到用户空间中。
+ write_size与read_size分别用来指定write_buffer与read_buffer的数据大小
+ write_consumed与read_consumed分别用来设定write_buffer与read_buffer中被处理的数据大小
+ 在前面的学习中，我们知道IPC数据与IPC应答数据由Handle、RPC代码、RPC数据、Binder协议构成。其中Handle、RPC代码、RPC数据保存在名称为binder_transaction_data的结构体中。如下图所示：  

![binder_transaction_data_ipc](http://7xn1yt.com1.z0.glb.clouddn.com/binder_transaction_data_ipc.png)

![binder_write_read_vs_binder_transaction_data](http://7xn1yt.com1.z0.glb.clouddn.com/binder_write_read_vs_transaction_data.png)

+ 如上图所示，binder_write_read的write_buffer指向含有Binder协议和binder_transaction_data的内存区域，在Binder Driver处理IPC数据时它们被作为一个处理单元。通常这种处理单元能被连续加载到write_buffer中，但为了说明的方便，只考虑有一个IPC数据的情形。同样，read_buffer指向由Binder协议与binder_transaction_data指向的内存区域

+ binder_write_read结构体中包含着用户空间中生成的IPC数据，Binder Driver也拥有一个相同的结构体。用户空间设置完binder_write_read结构体的数据后，调用ioctl()函数传递给Binder Driver，Binder Driver调用copy_from_user()函数将用户空间中的数据拷贝到自身的binder_write_read结构体中。相反，在传递IPC应答数据时，Binder Driver将调用copy_to_user()函数，将自身binder_write_read结构体中的数据拷贝到用户空间。

1.服务注册  

1.1 过程概览  

服务注册过程的示意图如下所示：  

![service_register](http://7xn1yt.com1.z0.glb.clouddn.com/service_register.png)

通过前面的博客我们知道，Context Manager先于其他所有Service Server进行，它最先使用Binder Driver，进入待机状态，之后等待Service Server的服务注册请求或者Client的服务检索请求。  
上图所示过程如下：  
1)Context Manager通过open()函数调用binder_open()函数，Binder Driver使用该函数为Context Manager生成并初始化binder_proc结构体;  

2)Context Manager在内核空间中开辟一块Buffer，用于接收IPC数据。Context Manager通过mmap()函数调用Binder Driver的binder_mmap()函数。Binder Driver使用binder_mmap()函数在内核空间中分配一块用于接收IPC数据的Buffer，其大小为调用mmap()函数时指定的大小，并被保存到binder_buffer结构体中。而后binder_buffer变量被注册到binder_proc结构体中;  

3)Context Manager通过ioctl()调用Binder Driver的binder_ioctl()函数。在binder_ioctl()函数中，Context Manager将处于待机状态，直到另一个进程（在服务注册阶段是指Service Server)向共发送IPC数据;  

4)Service Server通过open()调用Binder Driver的binder_open()函数，Binder Driver为Service Server生成binder_proc结构体，并将其初始化;  

5)Service Server通过mmap()调用Binder Driver的binder_mmap()函数.Binder Driver开辟一块用于接收IPC应答数据的Buffer，并将其保存到binder_buffer结构体中。而后Service Server生成IPC数据，用来调用Context Manager的服务注册函数;  

6)为了注册服务，Service Server将IPC数据传递给Binder Driver。Binder Driver将根据IPC数据中的Handle查找到Context Manager的binder_node结构体，以及binder_proc结构体;  

7)为Service Server要注册的服务生成binder_node结构体。Binder Driver将binder_node分别注册到Service Server的binder_proc结构体以及Context Manager的binder_proc结构体中;  

8)Binder Driver将在6)中查找到的Context Manager的binder_proc结构体中查找注册在binder_buffer结构体中的Buffer，并传递IPC数据;  

9)Binder Driver会记下IPC数据发送进程(Service Server)的binder_proc结构体，以便在Context Manager向Service Server发送应答数据时查找Serivce Server的binder_proc结构体;  

10)Binder Driver让Service Server处于待机状态，并将Context Manager从待机状态中唤醒，传递IPC数据。从待机状态中唤醒的Context Manager经由Binder Driver接收来自Service Server的IPC数据。Context Manager根据接收的IPC数据注册指定的服务，而后生成IPC应答数据(通知服务注册完成),并将其传递给Binder Driver;  

11)Binder Driver会在9)中的Service Server的binder_proc结构体中查找注册在binder_buffer结构体中的Buffer，并传送IPC应答数据，唤醒接收端的Service Server。Service Server被唤醒后，接收IPC应答数据，并根据接收的IPC应答数据进行相应处理。  

1.2 传递Binder数据的阶段1  

Context Manager调用binder_open()与binder_mmap()函数，初始化binder_proc结构体，创建用于接收IPC数据的binder_buffer结构体。  

在用户空间中生成一块Buffer，将read_buffer注册到binder_write_read结构体中，而后调用ioctl()函数，此时，binder_write_read结构体的read_size指的是用户空间的Buffer的大小。此时binder_ioctl()中调用的主要代码如下：  

    static long binder_ioctl(struct file*flip,unsigned int cmd,unsigned long arg){
        struct binder_proc*proc=flip->private_data;
        void __user*ubuf=(void __user*)arg;
        thread=binder_get_thread(proc);
        switch(cmd){
            case BINDER_WRITE_READ:
                struct binder_write_read bwr;
                if(copy_from_user(&bwr,ubuf,sizeof(bwr)))
                if(bwr.write_size>0){
                    ret=binder_thread_write(proc,thread,
                        (void __user*)bwr.write_buffer,bwr.write_size,
                        &bwr.write_consumed);
                }
                if(bwr.read_size>0){
                    ret=binder_thread_read(proc,thread,
                        (void __user*)bwr.read_buffer,bwr.read_size,&bwr.read_consumed,
                        flip->f_flags & O_NONBLOCK);
                }

                ...
        }
    }

binder_ioctl()函数的第二个参数cmd是ioctl()的命令，第三个参数arg是binder_write_read结构体，它是调用ioctl()函数时使用的参数。  

+ flip->private_data是在binder_open()函数中创建的binder_proc结构体
+ 由于proc的线程红黑树中还没有新建相应的线程，故在binder_get_thread()函数中新建一个binder_thread结构体
+ 分析ioctl()命令，前面已经说过，此时传递的命令是BINDER_WRITE_READ
+ 调用copy_from_user()方法将Context Manager持有的用户空间的binder_write_read结构体复制到内核空间中。而后根据read_size与write_size变量判断是发送IPC数据，还是接收IPC应答数据。Context Manager在接收IPC数据时，将生成相应的read_buffer，并且read_size的值将大于0
+ 由于read_size>0，故调用binder_thread_read()函数

1.2.1 binder_thread_read()分析  

binder_thread_read()中调用到的代码如下所示：  

    static int binder_thread_read(struct binder_proc *proc,
        struct binder_thread *thread,void __user*buffer,int size,
        signed long *consumed,int non_block)
    {
        ...
        ret=wait_event_interruptible_exclusive(proc->wait,
            binder_has_proc_work(proc,thread));
        ...
    }

Context Manager在binder_thread_read()函数中调用wait_event_interruptible_exclusive()函数。wait_event_interruptible_exclusive()函数通过当前进程的binder_proc结构体的wait变量将当前任务注册到待机队列中。而后将task_struct结构体内的state变量由TASK_RUNNING更改为TASK_INTERRUPTIBLE，再调用schedule()函数。当Context Manager的任务阙云太为TASK_INTERRUPTIBLE时，Context Manager就不再继续运行，它将进入待机状态，直至接收到IPC数据。  

1.3 传递Binder数据的阶段2  

这个阶段其实对应的是诸如MediaPlayerService::instantiate();这样的过程，只不过我们这里只从Binder Driver的角度考虑问题，所以只讨论ioctl()函数。  

这个阶段的示意图如下：  

![binder_flow_stage2](http://7xn1yt.com1.z0.glb.clouddn.com/binder_flow_stage2.png)

Service Server也会生成binder_write_read结构体，包含write_buffer与read_buffer两个成员变量。其中，read_buffer变量用于接收IPC应答数据，write_buffer变量用于发送IPC数据。binder_write_read结构体内容如下图所示：  

![binder_ipc](http://7xn1yt.com1.z0.glb.clouddn.com/register_ipc.png)

此时binder_ioctl()函数中执行的主要代码如下：  

    static long binder_ioctl(struct file*filp, unsigned int cmd,unsigned long arg)
    {
        struct binder_proc*proc=filp->private_data;
        void __user *ubuf=(void __user*)arg;
        thread=binder_get_thread(proc);
        switch(cmd){
            case BINDER_WRITE_READ:{
                struct binder_write_read bwr;
                if(copy_from_user(&bwr,ubuf,sizeof(bwr))){
                    ret=-EFAULT;
                    goto err;
                }
                if(bwr.write_size>0){
                    ret=binder_write(proc,thread,(void __user*)bwr.write_buffer,bwr.write_size,&bwr.write_consumed);
                }
                if(bwr.read_size>0)
                    ret=binder_thread_read(proc,thread,(void __user*)bwr.read_buffer,bwr.read_size,&bwr.read_consumed);
            }
        }
    }

+ Service Server调用binder_get_thread()函数生成binder_thread对象
+ 调用copy_from_user()函数，将调用空间的binder_write_read结构体拷贝至内核空间中。为了传递IPC数据，Service Server保存了binder_write_read结构体的write_buffer数据，故write_size>0
+ 由于write_size>0,故调用binder_thread_write()函数

binder_thread_write()函数的主要代码如下：  

    int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            void __user *buffer, int size, signed long *consumed)
    {
        uint32_t cmd;
        void __user *ptr = buffer + *consumed;
        void __user *end = buffer + size;
        while (ptr < end && thread->return_error == BR_OK) {
            if (get_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
           
            ...

            switch (cmd) {
        
            ...

            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;
                if (copy_from_user(&tr, ptr, sizeof(tr)))
                    return -EFAULT;
                ptr += sizeof(tr);
                binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                break;
            }
            
            ...

            }
            *consumed = ptr - buffer;
        }
        return 0;
    }

+ buffer+*consumed指向的是当前write_buffer中要处理的数据;buffer+size则指向缓冲区的末尾
+ 调用get_user()函数的目的是将IPC数据中的Binder协议拷贝至内核空间中。在服务注册时，所使用的Binder协议是BC_TRANSACTION
+ 调用copy_from_user()函数，将IPC数据中不包含Binder协议的binder_transaction_data结构体复制到内核空间中
+ 调用binder_transaction()函数，binder_transaction)函数使用binder_transaction_data结构体中的数据来执行Binder寻址、复制Binder IPC数据、生成及检索Binder节点等操作

binder_transaction()方法的主要代码如下：  

    static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply)
    {
        struct binder_transaction *t;
        struct binder_work *tcomplete;
        size_t *offp, *off_end;
        struct binder_proc *target_proc;
        struct binder_thread *target_thread = NULL;
        struct binder_node *target_node = NULL;
        struct list_head *target_list;
        wait_queue_head_t *target_wait;
        struct binder_transaction *in_reply_to = NULL;
        struct binder_transaction_log_entry *e;
        uint32_t return_error;
        e = binder_transaction_log_add(&binder_transaction_log);
        e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
        e->from_proc = proc->pid;
        e->from_thread = thread->pid;
        e->target_handle = tr->target.handle;
        e->data_size = tr->data_size;
        e->offsets_size = tr->offsets_size;
        if (reply) {
            in_reply_to = thread->transaction_stack;
            if (in_reply_to == NULL) {
                
                ...

            }
            binder_set_nice(in_reply_to->saved_priority);
            if (in_reply_to->to_thread != thread) {
               
                ...

                return_error = BR_FAILED_REPLY;
                in_reply_to = NULL;
                goto err_bad_call_stack;
            }
            thread->transaction_stack = in_reply_to->to_parent;
            target_thread = in_reply_to->from;
           
            ...

            if (target_thread->transaction_stack != in_reply_to) {
               
                ...

                return_error = BR_FAILED_REPLY;
                in_reply_to = NULL;
                target_thread = NULL;
                goto err_dead_binder;
            }
            target_proc = target_thread->proc;
        } else {
            if (tr->target.handle) {
                struct binder_ref *ref;
                ref = binder_get_ref(proc, tr->target.handle);
               
                ...

                target_node = ref->node;
            } else {
                target_node = binder_context_mgr_node;
                
                ...
            }
            e->to_node = target_node->debug_id;
            target_proc = target_node->proc;
            if (target_proc == NULL) {
                return_error = BR_DEAD_REPLY;
                goto err_dead_binder;
            }
            if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
                struct binder_transaction *tmp;
                tmp = thread->transaction_stack;
              
                ...

                while (tmp) {
                    if (tmp->from && tmp->from->proc == target_proc)
                        target_thread = tmp->from;
                    tmp = tmp->from_parent;
                }
            }
        }
        if (target_thread) {
            e->to_thread = target_thread->pid;
            target_list = &target_thread->todo;
            target_wait = &target_thread->wait;
        } else {
            target_list = &target_proc->todo;
            target_wait = &target_proc->wait;
        }
        e->to_proc = target_proc->pid;
        /* TODO: reuse incoming transaction for reply */
        t = kzalloc(sizeof(*t), GFP_KERNEL);
        
        ...

        binder_stats_created(BINDER_STAT_TRANSACTION);
        tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
       
        ...

        binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);
        t->debug_id = ++binder_last_id;
        e->debug_id = t->debug_id;
        
        ...

        if (!reply && !(tr->flags & TF_ONE_WAY))
            t->from = thread;
        else
            t->from = NULL;
        t->sender_euid = proc->tsk->cred->euid;
        t->to_proc = target_proc;
        t->to_thread = target_thread;
        t->code = tr->code;
        t->flags = tr->flags;
        t->priority = task_nice(current);
        t->buffer = binder_alloc_buf(target_proc, tr->data_size,
            tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
       
        ...

        t->buffer->allow_user_free = 0;
        t->buffer->debug_id = t->debug_id;
        t->buffer->transaction = t;
        t->buffer->target_node = target_node;
        if (target_node)
            binder_inc_node(target_node, 1, 0, NULL);
        offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));
        if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
           
            ...

        }
        if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
            
            ...

        }
        if (!IS_ALIGNED(tr->offsets_size, sizeof(size_t))) {
           
            ...

            return_error = BR_FAILED_REPLY;
            goto err_bad_offset;
        }
        off_end = (void *)offp + tr->offsets_size;
        for (; offp < off_end; offp++) {
            struct flat_binder_object *fp;
            
            ...

            fp = (struct flat_binder_object *)(t->buffer->data + *offp);
            switch (fp->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
                struct binder_ref *ref;
                struct binder_node *node = binder_get_node(proc, fp->binder);
                if (node == NULL) {
                    node = binder_new_node(proc, fp->binder, fp->cookie);
                    if (node == NULL) {
                        return_error = BR_FAILED_REPLY;
                        goto err_binder_new_node_failed;
                    }
                    node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
                    node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
                }
             
                ...

                ref = binder_get_ref_for_node(target_proc, node);
                if (ref == NULL) {
                    return_error = BR_FAILED_REPLY;
                    goto err_binder_get_ref_for_node_failed;
                }
                if (fp->type == BINDER_TYPE_BINDER)
                    fp->type = BINDER_TYPE_HANDLE;
                else
                    fp->type = BINDER_TYPE_WEAK_HANDLE;
                fp->handle = ref->desc;
                binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                           &thread->todo);
               
                ...

            } break;

            ...
            
            }
        }
        if (reply) {
            BUG_ON(t->buffer->async_transaction != 0);
            binder_pop_transaction(target_thread, in_reply_to);
        } else if (!(t->flags & TF_ONE_WAY)) {
            BUG_ON(t->buffer->async_transaction != 0);
            t->need_reply = 1;
            t->from_parent = thread->transaction_stack;
            thread->transaction_stack = t;
        } else {
            BUG_ON(target_node == NULL);
            BUG_ON(t->buffer->async_transaction != 1);
            if (target_node->has_async_transaction) {
                target_list = &target_node->async_todo;
                target_wait = NULL;
            } else
                target_node->has_async_transaction = 1;
        }
        t->work.type = BINDER_WORK_TRANSACTION;
        list_add_tail(&t->work.entry, target_list);
        tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
        list_add_tail(&tcomplete->entry, &thread->todo);
        if (target_wait)
            wake_up_interruptible(target_wait);
        return;

        ...

    }

+ 首先，由于tr->target.handle为0，故执行另一个分支，从而target_node=binder_context_mgr_node;即为Context Manager对应的binder_node，从而target_proc对应的是Context Manager的proc

+ 由于target_thread为NULL，故执行else中的内容，从而将binder_proc数据结构中的todo变量赋给target_list，以指定要执行的任务;同时将target_proc的等待队列赋给target_wait;

+ 变量t是一个指向binder_transactoin结构体的指针

IPC数据发送端与接收端通过struct binder_transaction来收发IPC数据，该结构体的定义如下：  

    struct binder_transaction {
        int debug_id;
        struct binder_work work;
        struct binder_thread *from;
        struct binder_transaction *from_parent;
        struct binder_proc *to_proc;
        struct binder_thread *to_thread;
        struct binder_transaction *to_parent;
        unsigned need_reply:1;
        /* unsigned is_dead:1; */   /* not used at the moment */
        struct binder_buffer *buffer;
        unsigned int    code;
        unsigned int    flags;
        long    priority;
        long    saved_priority;
        uid_t   sender_euid;
    };

+ t->from=thread;中的目的是查找发送IPC数据的进程，以便发送IPC应答数据，在收发Binder数据的第5个阶段中会用到

+ t->code=tr->code;目的是将IPC数据的RPC代码赋给binder_transaction结构体的code变量

+ binder_alloc_buf()函数的作用是分配一块空间，用于从接收端的IPC数据接收缓冲区中复制IPC数据。该接收缓冲空间由接收端在binder_mmap()函数中确定，binder_proc结构体的free_buffers指向该区域

+ copy_from_user(t->buffer->data,tr->data.ptr.buffer,tr->data_size);的作用是拷贝RPC数据到binder_buffer结构体的data变量中

**至此，Binder数据发送端设置binder_transaction结构体的过程结束了。接下来，接收端将从待机状态中苏醒。并从该结构体中获取数据。**

+ 调用binder_get_node()函数，在注册至binder_proc结构体中的Binder节点中，确认当前的Binder节点是否已经存在。由于首次注册，故不存在，从而新建节点。

+ 调用binder_new_node()函数不仅会创建新的binder_node，新生成的节点还会被注册到当前进程的binder_proc结构体中

+ 调用binder_get_ref_for_node()函数，检查指定的Binder节点是否已经存在于指定的binder_proc结构体中。若存在，则返回Binder节点所在的binder_ref结构体，否则新建binder_ref结构体并将指定的Binder节点注册其中，而后将其返回。另外,binder_get_ref_for_node()函数也将binder_ref结构体注册到binder_proc中的refs_by_node中

+ 接下来将BINDER_WORK_TRANSACTION赋值给t->work.type，这个值在第4个阶段会用到。当将type设置为BINDER_WORK_TRANSACTION时，Context Manager从binder_transaction结构体中获取Service Server传递而来的IPC数据，在第4阶段会详细讲解

+ 调用wake_up_interruptible()函数，并使用之前赋值过的target_wait作为参数，将Context Manager从待机状态中唤醒。在接下来的进程调度中，Context Manager将执行第4阶段的相应任务。

1.4 传递Binder数据的阶段3  

在传递Binder数据的第1个阶段中，Context Manager通过binder_write_read结构体的read_buffer进入到待机状态中。类似地，Service Server完成写入数据后，在binder_thread_read()中进入待机状态。Service Server将在收发Binder数据的第6个阶段中苏醒，接收IPC应答数据。  

1.5 传递Binder数据的阶段4  

Context Manager被唤醒后，将执行binder_thread_read()中接下来的动作。如下所示：  

    while(1){
        uint32_t cmd;
        struct binder_transacton_data tr;
        struct binder_work *w;
        struct binder_transaction *t=NULL;
        if(!list_empty(&proc->todo)&&wait_for_proc_work)
            w=list_first_entry(&proc->todo,struct binder_work,entry);
        switch(w->type){
            case BINDER_WORK_TRANSACTION:{
                t=container_of(w,struct binder_transaction,work);
            }break;
        }

    }

+ 调用list_first_entry()函数，查找Service Server在前面代码中注册的binder_work结构体，并将查找到的binder_work结构体返回。list_first_entry()函数是container_of()函数的引用函数，container_of()函数将根据首个参数查找到包含该参数的结构体的首地址。

+ 若binder_work结构体的type成员变量值为BINDER_WORK_TRANSACTION，则继续调用container_of()函数，查找持有binder_work结构体的binder_transaction结构体

之后，Context Manager通过Service Server创建的binder_transaction结构体获取IPC数据，并将IPC数据保存到binder_transaction_data结构体，以便将IPC数据传递到用户空间中，代码如下所示：  

    tr.code=t->code;
    tr.data_size=t->buffer->data_size;
    //code_1
    tr.data.ptr.buffer=(void*)t->buffer->data+proc->user_buffer_offset;

前面的代码都很好理解，关键是code_1处。内核空间接收Buffer的地址为t->buffer->data,而proc->user_buffer_offset则指用户空间的接收Buffer与内核空间的接收Buffer间的地址偏移。因此tr.data.ptr.buffer保存的是用户空间中接收Buffer的地址。binder_write_read结构体存在于用户空间中，read_buffer是binder_write_read结构体的成员变量，它可以引用保存在tr.data.ptr.buffer所指的用户空间中的IPC数据。在向接收端传递数据时,Binder IPC也采用相同的方法，将内核空间中的数据传递到用户空间中，而不需要调用copy_to_user()函数。通过这种方式，可以避免在内核空间与用户空间之间拷贝数据，从而提升IPC动作的效率。  

接下来，就要将binder_transaction_data中的数据传递到用户空间中。从该阶段开始，数据将被保存到Context Manager的read_buffer中，相应的代码如下：  

    cmd=BR_TRANSACTION;
    if(put_user(cmd,(uint32_t __user*)ptr))
        return -EFAULT;
    ptr+=sizeof(uint32_t);\
    if(copy_to_user(ptr,&tr,sizeof(tr)))
        return -EFAULT;
    if(cmd==BR_TRANSACTION && !(t->flags & TF_ONE_WAY)){
        t->to_parent=thread->transaction_stack;
        t->to_thread=thread;
        thread->transaction_stack=t;
    }

+ 首先设置Binder协议为BR_TRANSACTION,准备处理Context Manager的IPC数据

+ *ptr指向binder_write_read结构体的read_buffer成员变量，在传递Binder数据的第1个阶段中，Context Manager在进入待机状态前调用ioctl()函数时，它作为一个参数被传入函数中

+ *ptr指向binder_write_read结构体的read_buffer成员变量

+ 利用copy_to_user()将之前生成的binder_transaction_data结构体拷贝到read_buffer中

+ thread->transaction_stack=t;的目的是将当前的binder_transaction结构体注册到binder_thread结构体的transaction_stack中。binder_transaction结构体在传递Binder IPC数据的第5个阶段中用于查找IPC应答数据的接收端

1.6 传递Binder数据的阶段5  

在此阶段中，Context Manager将处理接收到的IPC数据，而后向Service Server发送IPC数据，告知处理已经完成。该阶段 数据传递过程与传递Binder数据的第2个阶段类似，涉及到的Binder协议分别是BC_REPLY(Contxt Manager-->Binder Driver)和BR_REPLY(Binder Driver-->Context Manager).  

    if(reply){
        in_reply_to=thread->transaction_stack;
        thread->transaction_stack=in_reply_to->to_parent;
        target_thread=in_reply_to->from;
        target_proc=target_thread->proc;
    }

+ 首先由thread->transaction_stack获取前面刚刚讲解过的注册的binder_transaction结构体

+ 之后由in_reply_to->from获取Service Server的binder_thread结构体

+ 由binder_thread即可获取Service Server的binder_proc，从而可以获得Service Server的接收端Buffer.之后将IPC应答数据保存到binder_transaction中。然后Service Server从待机状态中苏醒，查找相应的结构体，将IPC应答数据传递到用户空间中。由于该吉祥平安福且贵与传递Binder数据的阶段2类似，故不再赘述。

1.7 传递Binder数据的阶段6  

Service Server从待机状态中苏醒后，其行为动作与传递Binder数据第4个阶段类似，不同的是所使用的Binder协议为BR_REPLY.Service Server接收完IPC应答数据后将在用户空间中进行一系列与IPC应答数据相关的动作。

2.服务检索阶段  

在服务检索阶段，调用binder_ioctl()函数，将服务客户端的IPC数据发送到Context Manager，并将Context Manager的IPC应答数据发送到客户端。  

2.1 传递Binder数据的阶段1  

该阶段类似注册服务时传递Binder数据的第1个阶段，Context Manager进入待机状态，等待接收IPC数据。  

2.2 传递Binder数据的阶段2  

在该阶段中Binder Driver的行为动作类似于注册服务时传递Binder数据的第2个阶段。但是发送IPC数据的进程不是Service Server，而是服务客户端。  
服务客户端并不注册服务，所以在其RPC数据中不会存在flat_binder_object结构体。在服务客户端生成的IPC数据中包含Context Manager的Handle(0),RPC代码(服务检索函数)以及请求的服务名称。  
需要注意的是，由于IPC数据中不包含flag_binder_object对象，所以不会创建binder_node对象。其他动作与服务注册传递Binder的阶段2是一样的。  

2.3 传递Binder数据的阶段3  
与Service Server类似，服务客户端既发送IPC数据，又接收IPC应答数据，它会使用binder_write_read结构体的write_buffer与read_buffer两个成员变量。同服务注册阶段的传递IPC数据的第3个阶段一样，服务客户端将进入待机状态中，等待接收IPC应答数据。  

2.4 传递Bindder数据的阶段4  
该阶段与服务注册阶段的阶段4类似。Context Manager接收IPC数据后，会在服务目录中生成binder_object结构体，该结构体中包含着与服务名称相对应的服务目录编号。并且，将结构体中的type变量设置为BINDER_TYPE_HANDLE，生成IPC应答数据。binder_object会被复制到Binder Driver的flat_binder_object中，用来查找Binder节点。在下一个阶段中，通过binder_object数据，将Binder节点传递给服务客户端。  

2.5 传递Binder数据的阶段5  
在该阶段中，服务客户端将获取所用服务的Binder节点。在服务注册阶段,flat_binder_object结构体的type变量为BINDER_TYPE_BINDER，继而生成Binder节点。但是在服务检索阶段，type为BINDER_TYPE_HANDLE，Binder节点被传递给服务客户端，主要代码如下：  

    fp=(struct flat_binder_object*)(t->buffer->data+*offp);
    switch(fp->type){
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE:
            struct binder_ref*ref=binder_get_ref(proc,fp->handle);
            if(ref->node->proc==target_proc){
            }
            else{
                struct binder_ref*new_ref;
                new_ref=binder_get_ref_for_node(target_proc,ref->node);
                fp->handle=new_ref->desc;
            }break;
    }

+ 调用binder_get_ref()函数，其第一个参数proc为Context Manager的binder_proc，第二个参数fp->handle是服务客户端要使用的服务编号。服务编号即binder_ref结构体中desc变量的值。将desc的值传入binder_get_ref()函数后，即可获得与服务编号相对应的binder_ref结构体，此结构体由Context Manager持有

+ 调用binder_get_ref_for_node()函数，其中第一个参数target_proc是服务客户端的binder_proc结构体，第二个参数ref->node是ref对应的binder_node对象。在该函数内部，将为接收的binder_node结构体创建新的binder_ref结构体，并将其指向binder_node结构体

+ 修改RPC数据中flat_binder_object结构体内的服务编号，将其设置为当前binder_ref结构体的desc编号。而后在下一个阶段中，将包含修改后的RPC数据的IPC应答数据发送到服务客户端

2.6 传递Binder数据的阶段6  

该阶段与服务注册阶段的第6个阶段类似。服务客户端使用IPC应答数据中的Binder节点编号，进入服务使用阶段。  

3.服务使用阶段  

在服务使用阶段，服务客户端与拥有服务的Service Server进行IPC通信。通过在服务检索阶段获得的Binder节点编号，向Service Server发送IPC数据。在服务使用阶段中，发送IPC数据的进程是服务客户端，而接收端是Service Server.  

3.1 传递Binder数据的阶段1  

在服务使用阶段，Service Server进入待机状态，等待接收IPC数据。  

3.2 传递Binder数据的阶段2  

在服务使用阶段中，IPC数据包含与所用服务函数相关的RPC代码与RPC数据。在服务检索传递的Binder数据的第6个阶段中已经获取了服务的Binder节点编号，IPC数据中的Handle即是该Binder节点编号。  

该阶段与服务注册的传递Binder数据的第2个阶段类似，但是与binder_transaction()函数查找接收端binder_proc结构体的过程有所不同。  

    if(tr->target.handle){
        struct binder_ref*ref;
        ref=binder_get_ref(proc,tr->target.handle);
        target_node=ref->node;
    }else{
        target_node=binder_context_mgr_node;
    }
    target_proc=target_node->proc;

在服务注册阶段，Handle值为0，查找到的是Context Manager的binder_proc结构体;而在服务使用阶段则使用target.handle,查找对应进程的binder_proc结构体。  

+ 调用binder_get_ref()函数，查找desc值为指定Handle的binder_ref结构体。查找到指定的结构体后，将binder_ref结构体的node指向Service Server的binder_node结构体

+ binder_node结构体的proc指向Service Server的binder_proc结构体，即接收IPC数据的进程

3.3 传递Binder数据的阶段3  

该阶段与服务注册阶段类似，但接收IPC数据的不是Context Manager，而是Service Server. Service Server调用IPC数据中RPC代码所指的函数，并将RPC数据作为参数传入该函数中。此后的过程与Binder服务注册的传递Binder数据的第4，5，6阶段相同，不再赘述。  
