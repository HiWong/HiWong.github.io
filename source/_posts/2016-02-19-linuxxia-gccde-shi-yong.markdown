---
layout: post
title: "Linux下gcc的使用"
date: 2016-02-19 10:48:42 +0800
comments: true
categories: linux
---

1.gcc简介  

在正式使用gcc之前，我们先了解一下它的历史。  

现在的gcc(最早还只有C编译器，gcc其实只是GNU Compiler)是由Richard Stallman开发的，Stallman也是GNU的首创者，也是开源精神的布道者，著名的Linux开源项目就得益于GNU.  

随着计算机的发展，gcc逐渐开始支持C语言之外的语言，如C++,Object-C,Java,Fortan以及Ada等，详情可访问其主页 http://gcc.gnu.org  

现在，GNU工具链包括：  
1)GNU Compiler Collection(GCC):这里GCC只指编译器，包括链接等其它操作;  
2)GNU Make:编译和构建工程的自动化工具，即Makefile<!--more-->  
3)GNU Binutils:二进制工具，包括链接器和汇编器  
4)GNU Debugger(GDB):著名的gcc下调试工具  
5)GNU Autotools:可用于构建Makefile  
6)GNU Binson  

显然，gcc已经包含了十分强大的功能，它可以作为交叉编译工具，能够在当前CPU平台上为多种不同体系结构的硬件平台开发软件，非常适合在嵌入式领域的开发编译，如常用的arm-linux-gcc交叉编译工具。  

本文主要讨论Linux下gcc和g++的使用。  

实际上，我们使用的g++命令其编译过程调用的是与C语言gcc相同的工具，只不过链接过程有所不同。因此，如无特殊说明，gcc命令的使用也适用于g++.  

2.gcc安装  

实际上大多数Linux发行版都自带gcc，可通过gcc --version查看，如果考虑到gcc以及libc工具等，就可以直接安装build-essential，命令是：  

	$sudo apt-get install build-essential

当然也可以自己下载压缩包进行安装，只是更麻烦一些，也没有必要，此处不展开讨论。  

3.gcc的使用  

3.1 简单示例  
下面是一个C语言写的HelloWorld：  

	#include <stdio.h>
	int main(void){
		printf("Hello world!\n")

		return 0;
	}

保存main.c文件后，下面开始进行编译：  
	
	$gcc main.c
	$ls

然后会出现可执行文件a.out，由于没有指定可执行文件名，gcc默认为a.out，然后./a.out就可以看到Hello world!的输出了。  

实际上，利用gcc还可以得到编译过程中的中间文件，比如.s和.obj文件。在此之前，让我们先学习一下C代码生成可执行文件的过程，它的过程分为4步：预处理,编译,汇编,链接。如下图所示:  

![GCC_Compile_Step](http://7xn1yt.com1.z0.glb.clouddn.com/GCC_Compile_Step.png) 

3.2 gcc生成可执行文件的过程以及编译参数的使用  

3.2.1 还是上面HelloWorld的例子，-o选项表示生成相应的目标文件，如输入以下命令：  

	$gcc main.c -o main

可以生成可执行文件main  

3.2.2 -E选项表示预处理操作，预处理就是将宏定义展开，头文件展开。预处理之后的目标文件保存在main.i，如输入以下命令：  

	$gcc -E main.c -o main.i

生成的main.i文件如下所示(由于文件太长，只展示部分):   

	# 1 "/usr/lib/gcc/x86_64-linux-gnu/4.8/include/stddef.h" 1 3 4
	# 212 "/usr/lib/gcc/x86_64-linux-gnu/4.8/include/stddef.h" 3 4
	typedef long unsigned int size_t;
	# 34 "/usr/include/stdio.h" 2 3 4

	# 1 "/usr/include/x86_64-linux-gnu/bits/types.h" 1 3 4
	# 27 "/usr/include/x86_64-linux-gnu/bits/types.h" 3 4
	# 1 "/usr/include/x86_64-linux-gnu/bits/wordsize.h" 1 3 4
	# 28 "/usr/include/x86_64-linux-gnu/bits/types.h" 2 3 4


	typedef unsigned char __u_char;
	typedef unsigned short int __u_short;
	typedef unsigned int __u_int;
	typedef unsigned long int __u_long;


	typedef signed char __int8_t;
	typedef unsigned char __uint8_t;
	typedef signed short int __int16_t;
	typedef unsigned short int __uint16_t;
	typedef signed int __int32_t;
	typedef unsigned int __uint32_t;

	typedef signed long int __int64_t;
	typedef unsigned long int __uint64_t;


3.2.3 -S选项表示编译操作，将生成汇编文件(*.s文件),如输入如下命令：  

	$gcc -S main.c -o main.s

生成的main.s文件如下：  

	        .file   "main.c"
	        .section        .rodata
	.LC0:
	        .string "Hello world!"
	        .text
	        .globl  main
	        .type   main, @function
	main:
	.LFB0:
	        .cfi_startproc
	        pushq   %rbp
	        .cfi_def_cfa_offset 16
	        .cfi_offset 6, -16 
	        movq    %rsp, %rbp
	        .cfi_def_cfa_register 6
	        movl    $.LC0, %edi
	        call    puts
	        movl    $0, %eax
	        popq    %rbp
	        .cfi_def_cfa 7, 8
	        ret 
	        .cfi_endproc
	.LFE0:


值得注意的是由于CPU不同，即使是同样的C代码，在不同的平台上得到的汇编结果也不同，所以上面这个文件只是一种参照。在DSP,ARM等嵌入式平台上，使用gcc编译得到汇编，再根据汇编代码进行优化是一种常用的方法。  

3.2.4 -c选项将源文件生成目标文件main.obj，而目标文件已经是一种近似可执行文件了，通过链接操作链接相应的库就可以执行了。如输入以下命令：  

	$gcc -c main.c -o main.obj

归纳起来，上面分别使用了cpp预处理，gcc编译,as汇编.可见gcc是一套从预处理，编译，汇编到链接工具的集合。  

3.2.5 其它一些重要的选项参数说明  
1)-C只能配合-E使用，告诉预处理器不要丢弃注释:  

	$gcc -E -C main.c -o main.i

2)-M告诉预处理器输出一个适合make的规则，用于描述各目标文件的依赖关系，如下:  

	$gcc -M main.c -o main_makerule

打开main_makerule可看到其内容如下:  

	main.o: main.c /usr/include/stdc-predef.h /usr/include/stdio.h \
	 /usr/include/features.h /usr/include/x86_64-linux-gnu/sys/cdefs.h \
	 /usr/include/x86_64-linux-gnu/bits/wordsize.h \
	 /usr/include/x86_64-linux-gnu/gnu/stubs.h \
	 /usr/include/x86_64-linux-gnu/gnu/stubs-64.h \
	 /usr/lib/gcc/x86_64-linux-gnu/4.8/include/stddef.h \
	 /usr/include/x86_64-linux-gnu/bits/types.h \
	 /usr/include/x86_64-linux-gnu/bits/typesizes.h /usr/include/libio.h \
	 /usr/include/_G_config.h /usr/include/wchar.h \
	 /usr/lib/gcc/x86_64-linux-gnu/4.8/include/stdarg.h \
	 /usr/include/x86_64-linux-gnu/bits/stdio_lim.h \
	 /usr/include/x86_64-linux-gnu/bits/sys_errlist.h

实际上，在Android framework中大量的使用了这种方法生成的make文件。  

3)-w禁止各种类型的警告，不推荐使用;  
4)-Wall显示各种类型的警告，推荐使用，可在编译的过程中看到警告信息以便优化代码;  
5)-v显示编译过程的详细信息，Verbos的缩写;  
6)-llibrary连接名为library的库文件;  
7)-ldir在头文件的搜索路径列表中添加dir目录;  
8)-Ldir在'-llibrary'选项的搜索路径下添加dir目录  

4.多文件编译  
下面我们将写一个多文件组成的小程序。首先是multiply.h:  

	#ifndef MULTIPLY_H
	#define MULTIPLY_H

	extern int multiply(int a,int b);

	#endif

然后是multiply.c文件:  

	#include "multiply.h"

	int multiply(int a,int b)
	{
		return a*b;
	}

最后是main.c:  

	#include <stdio.h>
	#include "multiply.h"

	int main(void)
	{
		int a=5;
		int b=6;
		int c=multiply(a,b);

	#ifdef _DEBUG
		printf("c=%d\n",c);
	#endif

		return 0;
	}

下面是使用gcc进行编译的命令：  

	$gcc -D_DEBUG main.c multiply.c -o main
	$./main

执行后可看到c=30的输出  

上面未将add.h添加到编译文件是因为C语言的编译是以.c文件为单位的，每个.c文件都会编译对应到一个.s和.obj文件，而.h文件不会。在上面的例子中，我们只要保证在main.c在编译的时候能够找到multiply函数(会通过#include "multiply.h"实现)。在链接的时候，main.obj会自动找到multiply.obj中的multiply符号。  

5.共享库与静态库  
库是编译好的目标文件的一个打包，在链接时被加载到程序中，分为共享库和静态库。如之前使用到的printf定义就在libc中，我们只要将studio.h就能使用了(其实还要在gcc中使用-library选项，只不过gcc默认包含了该选项)。下面是静态库和共享库的定义。  

静态库:在Linux下是.a文件，在Windows下是.lib文件。在链接时将用户程序中使用到的外部函数机器码拷贝到程序可执行区  

共享库:在Linux下是.so文件，在Windows下是.dll文件。在链接时，只在程序可执行区建立一个索引表，而不拷贝机器码。在程序执行时，才根据索引表加载外部函数的机器码。共享库相对于静态库的优点是减少了可执行代码量的大小，同时共享库可同时被多个运行程序调用，能减小内存空间。  

无论是静态库还是共享库，在gcc选项中都要使用-I和-L分别指定库名和库路径。  
在上面multiply程序的基础上，先增加divide函数,下面是divide.h文件:  

	#ifndef DIVIDE_H
	#define DIVIDE_H

	extern int divide(int a,int b);

	#endif

然后是divide.c文件:  

	#include "divide.h"

	int divide(int a,int b)
	{
		return a/b;
	}

最后是main.c文件:  

	#include <stdio.h>
	#include "divide.h"
	#include "multiply.h"

	int main(void)
	{
		int a=20;
        int b=3;
        int c=multiply(a,b);
        int d=divide(a,b);

     #ifdef _DEBUG
        printf("c=%d\n",c);
        printf("c=%d\n",d);
     #endif

        return 0;
	}

静态库的生成需要使用到gcc和ar工具:  

	$gcc -c multiply.c divide.c
	$ar -r libmath.a multiply.o divide.o

然后是使用-l将静态库进行链接，由于是静态库，所以要添加-static选项;另外，前面说过,-Ldir的作用是在'-llibrary'选项的搜索路径列表中添加dir目录，故-L./的作用就是将当前路径增加到库路径中，以便链接时能够找到libmath。还有需要注意的一点是链接时是-lmath而不是-llibmath，因为这是Linux的规定，所有库文件(不管是共享库还是静态库)都需要在名称前加上lib.  

	$gcc -D_DEBUG main.c -lmath -static -L./ -o main

之后可得到带有调试信息的可执行文件main.  

仍以上面的程序为例，生成共享库的命令如下:  

	$gcc -shared -fPIC multiply.c divide.c -o libmath.so
    $gcc -D_DEBUG main.c -lmath -L./ -o main

其中-shared表示共享库，-fPIC表示生成与位置无关的代码。生成main后如果直接执行会出现"./main:error while loading shared libraries: libmath.so: cannot open shared object file: No such file or directory".这是因为共享库是在程序运行时(而不是编译时)才会将机器码插入到程序中，解决方法很简单，如果想要永久生效，则可以将当前路径加入到/etc/ld.so.conf中然后运行ldconfig,或者将当前so文件拷贝到/usr/lib/下;如果只想本次登录生效，则可利用export命令将当前路径加入到LD_LIBRARY_PATH中，由于PWD命令可获取当前路径，故加入命令如下：  

	$export LD_LIBRARY_PATH=LD_LIBRARY_PATH:$PWD

执行上述命令后再执行./main即可看到输出的调试信息。  

6.库路径、文件路径与环境变量  

之前已经说过，-I可以指定头文件路径(对应include),-L指定库路径,-l指定库名。那么系统默认的头文件路径和库路径在哪呢？利用以下命令可以查看，在不同设备上可能有所不同。  

	$cpp -v

在编译时也可以利用以下命令查看头文件路径：  

	$gcc -v main.c -lmath -L. -o main

通过-v选项，可以查看默认的头文件路径。  

gcc提供以下几个默认的环境变量：  
1)PATH:可执行文件和动态库(.so,.dll)的搜索路径;  
2)CPATH:头文件路径  
3)LIBRARY_PATH:库搜索路径，包括动态库和静态库  


