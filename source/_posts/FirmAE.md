---
title: FirmAE
date: 2022-04-22 15:33:52
tags:
    - iot工具
---

<!--more-->

# FirmAE

今天搞了半天的FirmAE的安装，可以记录一下其中的坑。

## 下载安装

### github

> https://github.com/pr0v3rbs/FirmAE/

根据github的md文档提示，安装命令只有三条，跟着做就好

```
git clone --recursive https://github.com/pr0v3rbs/FirmAE
./download.sh
./install.sh
```

### download报错解决

但是重点是这个download.sh，我反正运行了一个多小时都是报错的，就是各种443端口关闭或者访问不到，在网上搜集了好久的资料,发现有两种方法

#### 方法

1. **挂梯子**。网上师傅推荐就是**用proxychains跑脚本。**但是本人虚拟机很拉，没打算还再去搞梯子，于是还有第二种方法。
2. **手工下载**

download脚本

```
#!/bin/sh

set -e

download(){
 wget -N --continue -P./binaries/ $*
}

echo "Downloading binaries..."

echo "Downloading kernel 2.6 (MIPS)..."
download https://github.com/pr0v3rbs/FirmAE_kernel-v2.6/releases/download/v1.0/vmlinux.mipsel.2
download https://github.com/pr0v3rbs/FirmAE_kernel-v2.6/releases/download/v1.0/vmlinux.mipseb.2

echo "Downloading kernel 4.1 (MIPS)..."
download https://github.com/pr0v3rbs/FirmAE_kernel-v4.1/releases/download/v1.0/vmlinux.mipsel.4
download https://github.com/pr0v3rbs/FirmAE_kernel-v4.1/releases/download/v1.0/vmlinux.mipseb.4

echo "Downloading kernel 4.1 (ARM)..."
download https://github.com/pr0v3rbs/FirmAE_kernel-v4.1/releases/download/v1.0/zImage.armel
download https://github.com/pr0v3rbs/FirmAE_kernel-v4.1/releases/download/v1.0/vmlinux.armel

echo "Downloading busybox..."
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/busybox.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/busybox.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/busybox.mipsel

echo "Downloading console..."
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/console.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/console.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/console.mipsel

echo "Downloading libnvram..."
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/libnvram.so.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/libnvram.so.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/libnvram.so.mipsel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/libnvram_ioctl.so.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/libnvram_ioctl.so.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/libnvram_ioctl.so.mipsel

echo "Downloading gdb..."
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/gdb.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/gdb.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/gdb.mipsel

echo "Downloading gdbserver..."
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/gdbserver.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/gdbserver.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/gdbserver.mipsel

echo "Downloading strace..."
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/strace.armel
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/strace.mipseb
download https://github.com/pr0v3rbs/FirmAE/releases/download/v1.0/strace.mipsel

echo "Done!"

```

它作用就是去这几个网页把这些东西下下来，我花了五分钟下了所有东西，比我早上花了一个多小时还在那萌蠢地解决端口问题强多了……

这脚本就是把东西（https://github.com/pr0v3rbs/FirmAE/releases）都下到**binary目录**，但是它端口连不上，**约等于坑**

### 安装SquashFS images

DLink固件和SquashFS格式，很多环境不配置这个，binwalk无法正确解包

```
$ sudo apt-get install zlib1g-dev liblzma-dev liblzo2-dev
$ git clone https://github.com/devttys0/sasquatch
$ (cd sasquatch && ./build.sh)
```

```
若报错的话：
/usr/bin/ld: unsquash-1.o:/home/kali/Desktop/IoT/sasquatch-master/squashfs4.3/squashfs-tools/error.h:34: multiple definition of `verbose'; unsquashfs.o:/home/kali/Desktop/IoT/sasquatch-master/squashfs4.3/squashfs-tools/error.h:34: first defined here
verbose is extern in sasquatch/squashfs4.3/squashfs-tools/error.h
add "int verbose;" in unsquashfs.c
运行该命令：
sudo CFLAGS=-fcommon ./build.sh
```

## 实际使用

[固件下载链接](https://ftp.dlink.ru/pub/Router/)

### 固件下载

```

sudo wget https://ftp.dlink.ru/pub/Router/DIR-320/Firmware/old/
DIR-320A1_FW121WWb03.bin
```

### 提取

```
binwalk -Me DIR-320A1_FW121WWb03.bin
```

### 打包(可不做)

对文件夹重新打包成zip，从linux下转到windows下源码审计

```
zip -r -q -o pack.zip squashfs-root
```

上面命令将目录 squashfs-root下的所有文件，打包成一个压缩文件 pack.zip

### 启动FirmAE

到firmae下启动

```
root@attifyos:/home/iot/Documents/FirmAE# sudo ./init.sh
```

![image-20220422164647196](https://s2.loli.net/2022/04/22/KktzoI8JdOlXGcE.png)

再运行

```
sudo ./run.sh -r dir320 ./firmwares/DIR-320A1_FW121WWb03.bin
```

这个命令我估计是进入debug模式运行bin文件

![image-20220422165148712](https://s2.loli.net/2022/04/22/vGQil6zKZpO2cR4.png)

中途运行到开启network的时候启动比较慢，误以为是没连接上

可以看到network启动了，并且开放了一个ip：192.168.0.1

![image-20220422165427121](https://s2.loli.net/2022/04/22/Cbnu5haXzHk8KB4.png)

可以看到就已经复现漏洞环境了。
