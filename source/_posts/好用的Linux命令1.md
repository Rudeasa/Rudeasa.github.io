---
title: 好用的Linux命令1
date: 2022-03-07 17:08:17
tags:
    --Linux命令
---

<!--more-->

# 分享自己觉得很好用的Linux命令

## 1. 虚拟机重启网卡命令

```

sudo dhclient -r  

sudo dhclient

```

依次输入这两个命令可以解决很多时候连不上网的情况！

## 2.Ubuntu切换python版本命令

```
update-alternatives --install /usr/bin/python python /usr/bin/python2 100
 update-alternatives --install /usr/bin/python python /usr/bin/python3 150
 sudo update-alternatives --config python
```

三条命令真的非常好用，我曾因为pip与python版本补不匹配的情况导致我下不了很多python模块，也为此浪费了很多时间去搜集解决方法，终于发现了有这样的命令，有这三条命令可以灵活自如切换python，再也不用担心python版本兼容问题啦！

（**PS：第一条命令是给python2定义 第二条命令是给python3定义，第三条命令就是可以自己选择自己定义的python版本，必须按顺序输入命令~**）

## 3.进程命令

```
查看进程
ps -e | grep apt-get
杀死进程
sudo kill 进程号
```

曾在操作系统这门课上，因为几个进程再循环往复的运行导致我的程序报错，进程命令可以有效地控制进程的运行，也非常好用。
