---
title: ARM与MIPS
date: 2022-04-30 16:15:38
tags:
    - 逆向

---

<!--more-->

# 什么是ARM

ARM架构，称进阶精简指令集机器（Advanced RISC Machine）更早称作Acorn RISC Machine,是一个32位精简指令集（RISC）处理器架构。

## ARM大小端

最高有效位MSB（Most Significant Bit）对应大端(Big-endian)

最低有效位LSB（Least Significant Bit）对应大端(Little-endian)

- armel:arm eabi little endian的缩写，软件浮点
- armhf:arm  hard float 的缩写，硬件浮点
- arm64:64位的arm默认就是hf的，因此不需要hf的后缀

## ARM运行模式

1. 用户模式（USR）：ＡＲＭ处理器正常程序执行状态
2. 快速中断模式（FIQ）：高速数据传输或通道处理
3. 外部中断模式（IRQ）：通用的中断处理
4. 管理模式（SVC）：操作系统使用的保护模式
5. 数据访问终止模式（ABT）：当数据或指令预取终止时进入该模式，可用于虚拟存储及存储保护
6. 系统模式（SYS）：运行具有特权的操作系统任务
7. 未定义指令终止模式（UND）：未定义的指令执行是进入该模式

## ARM工作状态寄存器

![image-20220428165356204](https://s2.loli.net/2022/04/28/LVvX6Opf53AGeBk.png)

当系统（system）/用户（user）在运行的过程中遭遇突发状况（例如FIQ），那么数据会从CPSR保存到SPSR，等状况解决之后，再重新把数据返回给CPSR。

## ARM指令特点

- 所有指令定长：4字节/32位
- 大部分指令都可以在一个始终周期内执行完毕
- 每一个指令都可以有条件执行
- 指令在通电后可以配置为低位或高为优先
- LOAD/STORE模型，专有指令才能访问内存
- ARM两种指令格式：ARM&Thumb(16位)

## ARM环境搭建

安装交叉编译链接工具

```
sudo apt-get install gcc-arm-linux-gnueabi g++-arm-linux-gnueabi

sudo apt-get install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf
```

### 编译方式

```
动态：arm-linux-gnueabi-gcc hello.c -o hello_dym
```



# 什么是MIPS

MIPS（无互锁流水线微处理器）是一种精简（RISC）指令集体结构（ISA），起源于Stanford 大学John Hennessy教授的研究成果。

   

## MIPS结构的基本特点

1. 所有指令都是32位编码
2. 有些指令有32位目标地址编码；有些只有16位。因此要想加载任何一个32位值，就得用两个加载指令。16位的目标地址意味着，指令的跳转或子函数的位置必须在64k以内（上下32k）
3. 所有的动作原理上要求必须在1个时钟周期内完成，一个动作一个阶段
4. 有32个通用寄存器，每个寄存器32位（对32位机）或64位（对64位机）
5. 所有运算都是基于32位的，没有对字节和对半字的运算（MIPS里，字定义为32位，半字定义为16位）
6. 没有单独的栈指令，所有对栈的操作都是统一的内存访问方式
7. 由于MIPS固定指令长度，所以造成其编译后的二进制文件和内存占用空间比x86的要大
8. 寻址方式：只有一种内存寻址方式，基地址加上16位的地址偏移
9. 内存中的数据访问必须严格对齐（至少4字节对齐）
10. 跳转指令只有26位目标地址，再加上2位的对齐位，可寻址28位的空间，即256M
11. 条件分支指令只有16位跳转地址，加上2位的对齐位，共18位寻址空间，即256k
12. MIPS默认不把子函数的返回地址（就是调用函数的受害指令地址）存放到栈中，而是存放到￥31寄存器中
13. 流水线效应，由于采用了高度的流水线，有分支延迟效应和载入延迟效应

## MIPS大小端

MIPS：big-endian的MIPS架构

MIPSEL：little-endian的mips架构

## MIPS寄存器

![image-20220428173536726](https://s2.loli.net/2022/04/28/xgp98zeWDKaYydt.png)

一般情况下，32位处理器中每个寄存器的大小是32位，即4字节。

- r0:始终为0
- r1：为汇编程序保留
- r2-r3：用于存储返回结果
- r4-r7：用于存储参数
- r8-r15：存储临时数据
- r16-r23:保存以供以后使用的内容
- r24-r25:存储临时数据
- r26-r27:给操作系统使用
- r28:全局变量
- r29-r30:堆栈使用
- r31:返回地址

## MIPS编译运行

### 环境搭建

环境前提要安装qemu，这个之前写过就不写了

安装交叉编译链接工具

MIPS：

```
sudo apt-get install linux-libc-dev-mips-cross 
sudo apt-get install libc6-mips-cross libc6-dev-mips-cross 
sudo apt-get install binutils-mips-linux-gnu gcc-mips-linux-gnu
sudo apt-get install g++-mips-linux-gnu
```

MIPSEL：

```
sudo apt-get install linux-libc-dev-mipsel-cross 
sudo apt-get install libc6-mipsel-cross libc6-dev-mipsel-cross 
sudo apt-get install binutils-mipsel-linux-gnu gcc-mipsel-linux-gnu
sudo apt-get install g++-mipsel-linux-gnu
```

### 操作实例

**hello-static.c**

```
#include<stdio.h>
int main()
{
	printf("Hello mipsel Static\n");
	return 0;
}
```

#### 动态编译

通过命令` mipsel-linux-gnu-gcc hello-static.c -o hello-static`编译得到动态编译文件

![image-20220430103054257](https://s2.loli.net/2022/04/30/2Mnhf73Wa8XCHAB.png)

此时通过命令`qemu-mipsel ./hello-static`调试报错

![image-20220430103214361](https://s2.loli.net/2022/04/30/Is4enZLX15ON67r.png)

 正确用法运用 `-L`命令连接 `/usr`文件夹里的动态链接库mipsel-linux-gnu的lib库

![image-20220430103831724](https://s2.loli.net/2022/04/30/2YbwqsnpU3Etd1B.png)

使用命令 `qemu-mipsel -L /usr/mipsel-linux-gnu/ ./hello-static`，运行成功：![image-20220430103435565](https://s2.loli.net/2022/04/30/8Dx9eCuIHNEKqJk.png)

#### 静态编译

通过命令` mipsel-linux-gnu-gcc hello-static.c -o hello-static -static` 编译得到静态编译文件

![image-20220430103559169](https://s2.loli.net/2022/04/30/sYKbIjTaLhk269N.png)

运行命令`qemu-mipsel ./hello-static`，成功：

![image-20220430103643072](https://s2.loli.net/2022/04/30/7EP5ChSlJ6KgtrW.png)

