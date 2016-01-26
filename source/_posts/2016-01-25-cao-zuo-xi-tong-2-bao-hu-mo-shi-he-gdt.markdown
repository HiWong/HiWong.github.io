---
layout: post
title: "操作系统(2):保护模式和GDT"
date: 2016-01-25 16:41:02 +0800
comments: true
categories: OS
---

在8086/8088时代，CPU只有实模式这一种运行模式，本来16位的CPU只有2^16=64KB的寻址能力，但是通过20个地址总线以及"段：偏移"这样的处理，使其达到2^20=1MB的寻址能力.在实模式下，操作系统对于程序能够访问的地址没有任何限制，所以任意程序可以访问甚至修改任意地址的变量，显然这样容易发生严重的错误。  

到了80386时代，CPU具有32位寄存器，使其寻址能力达到2^32=4GB。为了更灵活地进行存储管理，并且对程序能够访问的物理地址进行限制，Intel引入了使用至今的保护模式.<!--more-->  

到了现在，计算机在刚启动时，运行在实模式下，启动到加载操作系统内核后就进入保护模式。保护模式有一些新的特色，用来增加多工和系统稳定度，比如内存保护，分页系统，以及硬件支持的虚拟内存等。大部分现今基于x86架构的操作系统都在保护模式下运行，包括Linux,FreeBSD以及Windows系统。  

实际上，到了80386时代，CPU有了四种运行模式，即实模式、保护模式、虚拟8086模式和SMM模式。本文只研究保护模式。  

1.保护模式下的寄存器  

在保护模式里80386扩展了8086的寄存器，原先的AX,BX,CX,DX,SI,DI,SP,BP从16位扩展到了32位，并改名为EAX,EBX,ECX,EDX,ESI,EDI,ESP,EBP,其中E就是Extend的意思。当然，保留了原先的16位寄存器的使用习惯，就像在8086下能用AH和AL访问AX的高低部分一样，不过EAX的低位部分能使用AX直接访问，高位却没有直接的访问方法，只能通过数据右移16位之后再访问。另外，CS,DS,ES,SS这几个16位段寄存器保留，并增加FS,GS两个段寄存器。另外还有其它很多新增加的寄存器，到后面我们会提到。  

2.保护模式下的内存分段  

我们知道，实模式下逻辑地址是由段值和段内偏移两部分组成。而在保护模式下，考虑到各种界限、属性、权限等信息，对一个内存段的描述需要8个字节，我们将其称为段描述符(Segment Descriptor).段描述符分为数据段描述符，指令段描述符和系统段描述符这3种。  

下面是一个段描述符的结构：  

![Segment Descriptor](http://7xn1yt.com1.z0.glb.clouddn.com/SegmentDescriptor.png)

从上图可以看到，段描述符所要表达的信息非常丰富，远不是32位段寄存器就能表示的，其主要由以下几部分组成：  
1)段基地址：用32位表示，可以从32位线性地址空间中的任何一个字节开始，不用像8086的13位表示那样必须被16整除；  
2)段界限：用20位表示，规定了段的大小；  
3)段属性如下  

	G:段界限的粒度，G=0表示段界限以1字节为单位，20位的段界限可表示1B到1M;G=1表示段界限以4K为单位，20位的段界限范围为4K-4G;
	DPL:描述符的特权级，用于特权检查以决定该段能否进行访问；
	DT:描述符的类型。DT=1表示当前是存储段的描述符，即程序可直接访问的代码和数据段；DT=0表示当前是系统段或门的描述符；
	TYPE:由四位组成，在数据段和代码段中的位2和位1的含义有所不同。分别表示为数据段还是代码段(E),是否一致代码段(C)或者数据段向低还是高扩展(ED),数据段是否可写(W)或代码段是否可读(R)、是否已被访问(A).

3.段描述符表  
   
显然，寄存器不足以存放所有内存段的描述符集合，所以这些描述符的集合(称为描述符表)被放置在内存里了。常用的描述符表有全局描述符表(Global Descriptor Table,GDT),局部描述符表(Local Descriptor Table,LDT),中断描述符表(Interrupt Descriptor Table,IDT),其中最重要的就是GDT,它为整个软硬件系统服务。  

由于GDT不能由GDT本身之内的段描述符进行描述，所以CPU使用一个专门的48位寄存器GDTR来保存GDT的信息，GDTR的48位分配图如下所示：  

        ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┓
        ┃           32位基地址             16位界限    
        ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━━━┛

显然，0-15位表示GDT的边界位置，16-47位存放的是GDT的基地址。既然用16位表示表的长度，由于2^16=65536，除以每一个描述符的8字节，显然最多能创建8192个描述符。  

4.如何进入保护模式  
 
前面说过，计算机刚启动时进入实模式，在加载操作系统内核之后进入保护模式，那到底是如何进入的呢？实际上很简单，在80386CPU的内部有5个32位的控制寄存器(Control Register,CR),分别是CR0-CR3,以及CR8。这些控制寄存器用来表示CPU的一些状态，其中的CR0寄存器的PE位(Protection Enable,保护模式允许位),0号位，就表示了CPU的运行状态，如果它为0代表实模式，为1则代表保护模式。通过修改这个位就可以立即改变CPU的工作模式。  

一旦CR0的PE位被修改，CPU就立即按照保护模式去建起了，所以这就要求我们必须在进入保护模式之前就在内存里放置好GDT，并且设置好GDTR寄存器。  

5. 段选择子  

前面讲到在保护模式下，段描述符替代了段值。但是一个段描述符有8个字节长，32位的段寄存器放不下，怎么办？  

答案就是在段寄存器里放段描述符在GDT或LDT中的索引，这也就是段选择子(Selector).就像寄存器放不下一整个数组或对象，我们就用寄存器保存指向它们的指针是一个道理。  

也就是说，在保护模式下，段寄存器(如CS)不再直接保存段基地址，但它也放不下一整个段描述符，所以我们用它保存段选择子，通过段选择子间接定位到段基地址。下面是段选择子的结构：  

        ┏━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┓
        ┃15┃14┃13┃12┃11┃10┃ 9┃ 8┃ 7┃ 6┃ 5┃ 4┃ 3┃ 2┃ 1┃ 0┃
        ┣━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━╋━━╋━━┻━━┫
        ┃                 描述符索引             ┃TI┃ RPL ┃         
        ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━┻━━━━━┛


		RPL(Requested Privilege Level): 请求特权级，用于特权检查。

		TI(Table Indicator): 引用描述符表指示位
		TI=0 指示从全局描述符表GDT中读取描述符；
		TI=1 指示从局部描述符表LDT中读取描述符。

从上图中可以看到，段选择子由三部分构成：  
1)段描述符索引：段选择子的高13位是段描述符索引，也就是段对应的描述符在描述符表中的序号。索引只有13位也是段描述符表最多只能包含8096个描述符的原因；  
2)TI:第二位是段描述符表的指示位(Table Indicator),TI=0表示从GDT中读取描述符；TI=1表示从LDT中读取；  
3)RPL:最低两位是讲求特权级(Requested Privilege Level)，用于特权检查。  

要注意的是段选择子仍然是保存在16位的段寄存器(如CS,DS)里。  

综上，在保护模式下，段寄存器中的段基地址加上偏移量就可以得到相应的物理地址。  

地址合成示意图如下：  

![protect_mode_address](http://7xn1yt.com1.z0.glb.clouddn.com/ProtectMode_Address.png)

6. 进入保护模式的详细过程及代码实现  

进入保护模式主要分为以下五步：  

1.准备GDT:准备好要跳转到的段描述符的段基地址，即LABEL_DESC_CODE32的基地址，初始时是0.我们模拟BIU,通过(CS<<4)+LABEL_SEG_CODE32计算出LABEL_SEG_CODE32代码段的真实物理，并将此物理地址拆分保存到LABEL_SEG_CODE32段描述符的基地址，其中AX和AL两行代码会保存到第2-4字节的段基址1，AH一行代码会保存到第7字节的段基址2；  

2.设置GDTR:同时，通过(CS<<4)+LABEL_GDT计算出GDT表基地址的真实物理地址，并将其保存到GdtPtr数据结构的高32位。注意GdtPtr与GDTR的结构是完全一致的，所以最后用LGDT指令将GdtPtr的值加载到GDTR中；  

3.打开A20:为了兼容8086,A20地址线关闭时地址超过1MB时会被回卷，所以必须打开A20来激活32位的寻址能力；  

4.设置CR0的PE位：寄存器CR0的第0位是PE位，此位为0时CPU运行于实模式，为1时运行于保护模式；  

5.跳转进入：现在可以通过选择子跳转了，因为这段代码位于16位的Section，所以要用jmp dword保证偏移量不会被截断。  

下面的汇编代码来自于渊的<<Orange’S操作系统的实现>>一书：  

	org 07c00h
    jmp LABEL_BEGIN

	[SECTION .gdt]
	;                                   段基址,         段界限, 属性
	LABEL_GDT:          Descriptor       0,                0, 0           ; 空描述符
	LABEL_DESC_CODE32:  Descriptor       0, SegCode32Len - 1, DA_C + DA_32; 非一致代码段
	LABEL_DESC_VIDEO:   Descriptor 0B8000h,           0ffffh, DA_DRW      ; 显存首地址

	GdtLen      equ $ - LABEL_GDT   ; GDT长度
	GdtPtr      dw  GdtLen - 1      ; GDT界限
	            dd  0               ; GDT基地址

	SelectorCode32      equ LABEL_DESC_CODE32   - LABEL_GDT
	SelectorVideo       equ LABEL_DESC_VIDEO    - LABEL_GDT

	[SECTION .s16]
	[BITS   16]
	LABEL_BEGIN:
	    mov ax, cs
	    mov ds, ax
	    mov es, ax
	    mov ss, ax
	    mov sp, 0100h

	    ; 1）初始化 32 位代码段描述符
	    xor eax, eax
	    mov ax, cs
	    shl eax, 4
	    add eax, LABEL_SEG_CODE32
	    mov word [LABEL_DESC_CODE32 + 2], ax
	    shr eax, 16
	    mov byte [LABEL_DESC_CODE32 + 4], al
	    mov byte [LABEL_DESC_CODE32 + 7], ah

	    ; 2）加载 GDTR
	    xor eax, eax
	    mov ax, ds
	    shl eax, 4
	    add eax, LABEL_GDT
	    mov dword [GdtPtr + 2], eax
	    lgdt    [GdtPtr]

	    ; 关中断
	    cli

	    ; 3）打开地址线A20
	    in  al, 92h
	    or  al, 00000010b
	    out 92h, al

	    ; 4）准备切换到保护模式
	    mov eax, cr0
	    or  eax, 1
	    mov cr0, eax

	    ; 5）真正进入保护模式
	    jmp dword SelectorCode32:0  ; 执行这一句会把 SelectorCode32 装入 cs,
	                                ; 并跳转到 Code32Selector:0  处

	[SECTION .s32]
	[BITS   32]
	LABEL_SEG_CODE32:
	    mov ax, SelectorVideo
	    mov gs, ax                  ; 视频段选择子
	    mov edi, (80 * 11 + 79) * 2 ; 屏幕第 11 行, 第 79 列。
	    mov ah, 0Ch                 ; 0000: 黑底    1100: 红字
	    mov al, 'P'
	    mov [gs:edi], ax

	    jmp $                       ; 到此停止

	SegCode32Len    equ $ - LABEL_SEG_CODE32


