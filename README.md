# Reverse-Engineering
逆向工程

汇编指令和逆向工程
1	基础知识
1.1	介绍
逆向工程（又称逆向技术），是一种产品设计技术再现过程，即对一项目标产品进行逆向分析及研究，从而演绎并得出该产品的处理流程、组织结构、功能特性及技术规格等设计要素，以制作出功能相近，但又不完全一样的产品。逆向工程源于商业及军事领域中的硬件分析。其主要目的是，在不能轻易获得必要的生产信息下，直接从成品的分析，推导出产品的设计原理。
软件逆向工程有多种实现方法，主要有三种：
1）分析通过信息交换所得的观察。
2）反汇编，即使用反汇编器，把程序的原始机器码，翻译成较便于阅读理解的汇编代码。
3）反编译，即使用反编译器，尝试从程序的机器码或字节码，重现高级语言形式的源代码。
本文主要对反汇编方法进行一些经验总结，下面先介绍一下寄存器的基础知识。
1.2	寄存器
1.2.1	通用寄存器（32位）
通用寄存器一共有八个：EAX、EBX、ECX、EDX、ESP、EBP、EDI、ESI。
其中EAX、EBX、ECX、EDX称为数据寄存器，用于存放计算过程中所用操作数、结果或其他信息。除了直接访问外，还可分别对其高十六位和低十六位，它们的低十六位就是把它们前边儿的E去掉，即EAX的低十六位就是AX。而且它们的低十六位又可以分别进行八位访问，也就是说，AX还可以再进行分解，即AX还可分为AH（高八位）AL（低八位）。
ESP、EBP、EDI、ESI四个寄存器主要用途就是在存储器寻址时，提供偏移地址，因此它们可以称为指针或变址寄存器。

EAX 是累加器(accumulator)，它是很多加法乘法指令的缺省寄存器。 
EBX 是基地址(base)寄存器，在内存寻址时存放基地址。 
ECX 是计数器(counter)，是重复(REP)前缀指令和LOOP指令的内定计数器。 
EDX 则总是被用来放整数除法产生的余数。 
ESP称为堆栈指针寄存器。堆栈是以“后进先出”方式工作的一个存储区，它必须存在于堆栈段中，因而其段地址存放于SS寄存器中。它只有一个出入口，所以只有一个堆栈指针寄存器。ESP的内容在任何时候都指向当前的栈顶。
EBP，它称为基址指针寄存器，它们都可以与堆栈段寄存器SS联用来确定堆栈中的某一存储单元的地址，ESP用来指示段顶的偏移地址，而EBP可作为堆栈区中的一个基地址以便访问堆栈中的信息。
ESI（源变址寄存器）和EDI（目的变址寄存器）一般与数据段寄存器DS联用，用来确定数据段中某一存储单元的地址。在很多字符串操作指令中，DS:ESI指向源串，而ES:EDI指向目标串。这两个变址寄存器有自动增量和自动减量的功能，可以很方便地用于变址。在串处理指令中，ESI和EDI作为隐含的源变址和目的变址寄存器时，ESI和DS联用，EDI和附加段ES联用，分别达到在数据段和附加段中寻址的目的。
1.2.2	专用寄存器
专用寄存器也叫控制寄存器，有两个，一个是EIP，一个是FLAGS。
EIP是指令指针寄存器，算是所有寄存器中最重要的一个了，它用来存放代码段中的偏移地址。在程序运行的过程中，它始终指向下一条指令的首地址。它与段寄存器CS联用确定下一条指令的物理地址。当这一地址送到存储器后，控制器可以取得下一条要执行的指令，而控制器一旦取得这条指令就马上修改EIP的内容，使它始终指向下一条指令的首地址。可见，计算机就是用EIP寄存器来控制指令序列的执行流程的。 那些跳转指令，就是通过修改EIP的值来达到相应的目的的。
FLAGS是标志寄存器，又称PSW(program status word)，即程序状态寄存器。这一个是存放条件标志码、控制标志和系统标志的寄存器。
OF overflow flag 溢出标志 操作数超出机器能表示的范围表示溢出，溢出时为1。
SF sign Flag 符号标志 记录运算结果的符号，结果负时为1。
ZF zero flag 零标志 运算结果等于0时为1，否则为0。
CF carry flag 进位标志 最高有效位产生进位时为1，否则为0。
AF auxiliary carry flag 辅助进位标志 运算时，第3位向第4位产生进位时为1，否则为0。
PF parity flag 奇偶标志 运算结果操作数位为1的个数为偶数个时为1，否则为0。
DF direcion flag 方向标志 用于串处理。DF=1时，每次操作后使SI和DI减小。DF=0时则增大。
IF interrupt flag 中断标志 IF=1时，允许CPU响应可屏蔽中断，否则关闭中断。
TF trap flag 陷阱标志 用于调试单步操作。
1.2.3	段寄存器
段寄存器一共六个，分别是CS代码段，DS数据段，ES附加段，SS堆栈段，FS以及GS这两个还是附加段。
CS（Code Segment）代码段寄存器
DS（Data Segment）：数据段寄存器
SS（Stack Segment）：堆栈段寄存器
ES（Extra Segment）：附加段寄存器
2	汇编指令
汇编指令语句的格式: [标号:] 指令助记符 [[目的操作数][, 源操作数]] [; 注释]
	指令助记符
如MOV, SUB这些词分别表示传送, 减法. 汇编源程序时, 系统使用内部对照表将每条指令的助记符翻译成对应的机器码
	目的操作数
目的操作数一共有两个作用：1）参与指令操作；2）暂时储存操作结果。
	源操作数
源操作数主要提供原始数据或操作对象, 面向所有寻址方式. 例如, 在指令SUB AX, BX 中 的值作为减数提供给指令SUB
	注释, 在汇编中用 ; 号, 后面的内容将被注释

关于汇编语言寄存器和指令操作的整理
https://www.cnblogs.com/technology/archive/2010/05/16/1736782.html
汇编语言的所有指令：
https://blog.csdn.net/baishuiniyaonulia/article/details/78504758
https://blog.csdn.net/Andrewniu/article/details/80566277
https://blog.csdn.net/bjbz_cxy/article/details/79467688
2.1	数据传送指令
	通用数据传送指令:
MOV     DST, SRC      ;传送指令: 把源操作数的内容送入目的操作数
PUSH     SRC        ;压栈指令: 将一个字数据压入当前栈顶, 位移量disp=-2的地址单元. 数据进栈时, 栈指针SP首先向低地址方向移动两个字节位置, 接着　　　　数据进栈, 形成新的栈顶
POP       DST              ;出栈指令: 弹出栈顶元素, 后将栈顶指针向栈底方向移动一个字
XCHG    OPR1, OPR2        ;交换指令: 将这两个操作数交换
	地址传送指令:
LEA        DST, SRC      ;装载有效地址指令: 该指令将源操作数的偏移量OA装载到目的操作数中
LDS        DST, SRC      ;装载数据段指针指令: 将当前数据段中的一个双字数据装入到一个通用寄存器SI(双字数据的低字)和数据段寄存器DS(双字数据的高字)中
	MOV与LEA的区别：
mov ax,BUFF ;是把BUFF这个内存单元中的数据放入到ax寄存器中，而lea ax,BUFF ;是把BUFF这个内存单元的地址放入到ax寄存器中。两者区别就是一个传递的是内容，一个传递的是地址。

word ptr 按字处理，两字节
byte ptr  按字节处理，一字节
dword   双字 就是四个字节
ptr     pointer缩写 即指针
[ ] 里的数据是一个地址值，这个地址指向一个双字型数据
比如mov eax, dword ptr [12345678]  把内存地址12345678中的双字型（32位）数据赋给eax
2.2	算术运算指令
	加法指令:
ADD      DST, SRC      ;DST+SRC的和存放到DST中去
INC       DST           ;增1指令
	减法指令:
SUB       DST, RSC      ;DST-SRC, 存放到DST中
SBB       DST, SRC      ;带借位减法指令, DST-SRC-CF
DEC       DST            ;减1指令
CMP      OPR1, OPR2     ;比较指令
	乘法指令:
MUL      SRC      ;无符号数乘指令, AL*SRC, 结果放入AX中
IMUL     SRC      ;有符号数乘指令, AL*SRC, 结果放入AX中
	除法指令:
DIV     SRC   ;无符号数除指令, AX/SRC, 商放入AL中, 余数放在AH中
IDIV       SRC      ;符号数除指令, AX/SRC, 上放入AL中, 余数放在AH中
2.3	逻辑运算指令
	ANL做AND（逻辑与）运算
	ORL做OR（逻辑或）运算
	XRL 做（逻辑异或）运算
	CLR 清除为0
	CPL 取反指令
	RL 不带进位左环移
	RLC 带进位左环移
	RR 不带进位右环移
	RRC 带进位右环移
	SHLD(双精度左移)
写法：SHLD REG16/REG32/MEM16/MEM32, REG16/REG32, IMM8/CL;(类型须匹配)
作用：将OPRD1的各二进制左移，并将oprd1的最高位移到CF,oprd2的最高位移到oprd1的最低位，但是，oprd2的值不变。
	SHRD（双精度右移）
写法与作用与双精度左移类似。注意移动方向为右移。
2.4	串操作指令
	MOVS   ;串传送指令
	CMPS    ;串比较指令
	SCAS     ;串扫描指令
	LODS     ;装入串指令
	STOS      ;存储串指令
2.5	控制转移指令
	转移指令:
JMP        ;无条件转移指令
JX          ;条件转移指令(JC/JNC, JZ/JNZ, JE/JNE, JS/JNS, JO/JNO, JP/JNP…)
	循环指令:
LOOP     标号    ; 
	条件循环指令:
LOOPZ/LOOPE, LOOPNZ/LOOPNE       ;前者用于找到第一个不为0的事件, 后者用于找到第一个为0的事件
	子程序调用指令:
CALL      子程序名       ;段内直接调用
RET
	中断指令:
INT        N(中断类型号)     ;软中断指令
IRET       ;中断返回指令

JL SHORT 004030B4
SHORT说明进行短转移，偏移地址不远

条件跳转(一般配合cmp使用)
JZ/JE(Jump if zero,or equal) 结果为零(或相等)则跳转    注：检测Z位
JNZ/JNE(Jump if not zero,or not equal) 结果不为零(或不相等)则跳转  注：检测Z位
JS(Jump if sign) 结果为负则跳转          注：检测S位
JNS(Jump if not sign) 结果为正则跳转     注：检测S位
JB(Jump Below):小于则跳转               注：检测C位
JNB(Jump Not Below ):大于或等于则跳转    注：检测C位
这些C位,Z位，S位在哪里看呢?
在OD工具中的寄存器窗口中认识了标志寄存器,如图:  
 
3	实例解析
3.1	LOCAL含义
MOV DWORD PTR SS:[LOCAL.37],EAX 
上述命令中LOCAL什么意思？
含义同：MOV DWORD PTR SS:[ESP+0A8],EAX
双击指令行即可看到转换后的命令：
 
在OD中可以通过反勾选下面红框里的选项来避免出现LOCAL，直接显示ESP+XXX的地址：
 
在OD里，[local.1] 是 ebp-4， [local.2] 是 ebp-8， 以每4个字节递增
参考：
https://zhidao.baidu.com/question/358796000.html
https://stackoverflow.com/questions/19909100/why-does-my-debugger-display-local-8-instead-of-ebp-20


3.2	TEST CL,CL的作用是什么
汇编指令test cl,cl的作用是什么？
https://zhidao.baidu.com/question/745117673594622332.html
https://zhidao.baidu.com/question/596001958.html?qbl=relate_question_0&word=TEST%20CL%2CCL%BB%E3%B1%E0
test执行的就是and的指令，只不过不会保存and执行的结果，而是根据and的结果设置flags寄存器的各种标志

3.3	MOV
lea eax,dword ptr ss:[esp-8]的含义？
将 esp - 8 的值放入 eax ，即取 ss:[esp-8] 的偏移地址
https://bbs.csdn.net/topics/230075420

MOV EAX,DWORD PTR SS:[EBP-1C] 是什么意思?
https://zhidao.baidu.com/question/371671974.html
BYTE 单字节、DWORD 双字节：8086CPU的指令，可以处理两种尺寸的数据，byte和word。所以在机器指令中要指明，指令进行的是字操作还是字节操作。
3.3.1	MOV FS
MOV EAX,DWORD PTR FS:[0]
FS寄存器指向当前活动线程的TEB结构（线程结构）
偏移 说明
000 指向SEH链指针
004 线程堆栈顶部
008 线程堆栈底部
00C SubSystemTib
010 FiberData
014 ArbitraryUserPointer
018 FS段寄存器在内存中的镜像地址
020 进程PID
024 线程ID
02C 指向线程局部存储指针
030 PEB结构地址（进程结构）
034 上个错误号

FS:[0] 指向的是TIB[Thread information Block]结构中的 ;EXCEPTION_REGISTRATION 结构
https://www.liangzl.com/get-article-detail-4142.html

3.4	实战DES算法注册机
[原创]实战DES算法注册机（破解教程）
https://bbs.pediy.com/thread-59285.htm
4	逆向工程
4.1	工具介绍
静态分析工具首推IDA，有了它其他的反汇编工具基本用不着。
动态调试工具有OD和windbg。 调试应用层程序两个调试器都可以，OD因为主要面向逆向，窗口布局更为合理直观且插件众多，所以一般情况下都首选OD，windbg没那么方便，大部分操作通过命令来进行，但它也有它的优势，各种命令（内置命令、元命令和扩展命令）提供了强大的控制和分析能力，所以windbg有时也会用到。如果要调试内核程序或模块那OD就无能为力了，windbg可以说是唯一的选择。

OllyDbg工具下载
http://3ms.huawei.com/km/blogs/details/1689563
OllyDBG完美教程(超强入门级)
https://blog.csdn.net/vicroad2014/article/details/82726003

4.2	OD简介
4.2.1	窗口介绍
 
反汇编窗口：显示被调试程序的反汇编代码，标题栏上的地址、HEX 数据、反汇编、注释可以通过在窗口中右击出现的菜单 界面选项->隐藏标题 或 显示标题 来进行切换是否显示。用鼠标左键点击注释标签可以切换注释显示的方式。

寄存器窗口：显示当前所选线程的 CPU 寄存器内容。同样点击标签 寄存器 (FPU) 可以切换显示寄存器的方式。
信息窗口：显示反汇编窗口中选中的第一个命令的参数及一些跳转目标地址、字串等。
数据窗口：显示内存或文件的内容。右键菜单可用于切换显示方式。
堆栈窗口：显示当前线程的堆栈。

4.2.2	常用快捷键
F2：设置断点，只要在光标定位的位置按F2键即可，再按一次F2键则会删除断点。
F8：单步步过。每按一次这个键执行一条反汇编窗口中的一条指令，遇到 CALL 等子程序不进入其代码。
F7：单步步入。功能同单步步过(F8)类似，区别是遇到 CALL 等子程序时会进入其中，进入后首先会停留在子程序的第一条指令上。
F4：运行到选定位置。作用就是直接运行到光标所在位置处暂停。
F9：运行。按下这个键如果没有设置相应断点的话，被调试的程序将直接开始运行。
CTR+F9：执行到返回。此命令在执行到一个 ret (返回指令)指令时暂停，常用于从系统领空返回到我们调试的程序领空。
ALT+F9：执行到用户代码。可用于从系统领空快速返回到我们调试的程序领空。
4.2.3	PE文件
PE文件的全称是Portable Executable，意为可移植的可执行的文件，常见的EXE、DLL、OCX、SYS、COM都是PE文件
4.3	OD使用技巧
4.3.1	设断点
在反汇编的Comments窗口点右键，选择Search for，All referenced strings
根据关键字查找对应的代码位置，找到后可以设置断点。
 
4.3.2	堆栈窗口快速查看ESP和某一地址之间差值
在堆栈窗口用鼠标双击某一行地址，即可看到前后其它地址和选中行之间的差值。
大大的方便了查看指令中对应的地址：
MOV DWORD PTR SS:[ESP+0A8],EAX
 
双击第一行之后变成了下面的样子，是不是很好找到指令中对应的行了：
 
4.3.3	使用Watch窗口
Watches界面可以跟踪地址的数据变化，使用方法参考OD包中的help.pdf
如，观察地址内容：[DWORD 0019C118]
4.3.4	使用Memory窗口
在Memory map界面使用Ctrl + B进行内存搜索
https://blog.csdn.net/cssxn/article/details/84678074
4.4	IDA使用技巧
开发了IDA的天才是Ilfak，他的个人博客有很多IDA的教程
https://www.hexblog.com/
 
4.4.1	IDA逆向技巧
IDA逆向技巧(持续更新)
https://blog.csdn.net/yuqian123455/article/details/82317425
IDA Pro7.0使用技巧总结
https://blog.csdn.net/qq_40351988/article/details/88107647
如何参考IDA来逆向代码
https://blog.csdn.net/yuqian123455/article/details/83187724
逆向代码
1、如果是小的代码点，我们只需要将代码看懂，使用高级语言实现就行了；
2、如果是大批量的反代码，我们需要做的是将函数含义，函数参数，返回值，数据结构先分析好，建好结构体，然后将整个函数转成.c 文件，再将函数拷贝到VS中，然后逐句重写；
IDA 逆向代码，分析清楚函数的参数，然后直接将IDA的代码导出来，然后复制粘贴，然后每行去逆向代码，这里面变量就是不变的。
4.4.2	IDA设置
在流程视图中添加地址偏移，这样取地址就非常方便，不再需要按空格切换视图去找，在菜单栏中设置：option-->general
同时打开自动注释也方便理解代码
 
4.4.3	IDA反编译器的使用
https://blog.csdn.net/lixiangminghate/article/details/83752367
生成代码的插件：Edit -> Plugins -> Hex-Rays Decompiler
IDA反编译器，正如IDA官网所说，虽然还原出来的代码不能直接使用，但是其参考作用不容否认。
生成伪代码：view-Open subviews-Generate pesudocode (快捷键F5)
如：
 
4.4.4	IDA与OD的联合使用
在OD中下断点，单步跟踪调试，找到关键代码位置，使用IDA看静态代码流程，需要注意的是IDA的需要与OD基地址对齐，这样才方便在IDA中查找代码位置，具体方式如下：
Edit --->  Segment ----- Rebase  Program  : 设置基地址；

在WINdbg中基地址在“符号”中查看；
在OD中基地址在Model中查看；
IDA中G + 地址，即可找对对应位置；
IDA的逆向分析水平就是一个逆向工程师的水平；
4.4.5	BYTE1  BYTE2含义
用hex ray逆向出的C代码有些不明白的符号：BYTE1  BYTE2  BYTE3  __SETO__等
https://bbs.csdn.net/topics/350257523
https://blog.csdn.net/huiguixian/article/details/52026710

__int64 __thiscall sub_402E00(_DWORD *this, unsigned __int64 a2)
{
    unsigned __int64 v2; // rt0

    v2 = a2 >> 16;
    return this[BYTE1(v2) + 769]
      + (this[BYTE2(a2) + 513] ^ (unsigned int)(this[(unsigned __int8)a2 + 1] + this[BYTE1(a2) + 257]));
}


宏定义：
#define BYTEn(x, n)   (*((_BYTE*)&(x)+n))

#define BYTE1(x)   BYTEn(x,  1)         // byte 1 (counting from 0)
#define BYTE2(x)   BYTEn(x,  2)

5	参考
5.1	汇编指令
破解入门（一）-----常用寄存器
https://blog.csdn.net/qiurisuixiang/article/details/7596507

5分钟看懂汇编语言
https://blog.csdn.net/qq_40663092/article/details/79852766

5.2	逆向工程
从零开始的程序逆向之路 第一章——认识OD(Ollydbg)以及常用汇编扫盲
https://www.cnblogs.com/ichunqiu/p/9355253.html
BackTrack逆向工程测试指导
http://3ms.huawei.com/hi/group/337/thread_3430791.html?mapId=1647875
安全技术入门----反汇编，教你玩破解
http://3ms.huawei.com/km/blogs/details/1538123
破解入门（四）-----实战"单步跟踪法"脱壳
https://blog.csdn.net/qiurisuixiang/article/details/7649591
图解Ollydbg简单逆向操作案例
https://blog.csdn.net/bcbobo21cn/article/details/51291025

