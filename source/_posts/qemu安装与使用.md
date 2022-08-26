---
title: qemu安装与使用
date: 2022-04-18 08:53:36
tags:
    - iot
---

<!--more-->

# 前言

​	这两天在复现cve的时候疯狂踩坑漏洞环境，本着我之前为数不多复现成功过的一个iot漏洞的情况，我依旧采用qemu来模拟环境运行漏洞环境，但是没想到还是踩了不少坑，决定认真学习一下系统地学习一下qemu的安装与使用。	

# qemu介绍

[官方文档]: https://www.qemu.org/docs/master/

qemu是一款可执行硬件虚拟化的虚拟机，与他类似的还有Bochs、PearPC，但qemu具有高速（配合KVM）、跨平台的特性
qemu主要有两种运行模式：qemu-user 和 qemu-system

1. 用户模式（User Mode)，亦称使用者模式。qemu能启动那些为不同中央处理器编译的Linux程序。
2. 模拟模式（System  Mode)，亦成为系统模式。qemu能模拟整个计算机系统，包括中央处理器及其他周边设备，它使能为跨平台编写的程序进行测试及排错工作变得容易。其亦能用来在一部主机上虚拟数个不同的虚拟计算机，类似我们平常使用的Vmare、VirtualBox等。

# Ubuntu安装

- apt-get安装

   `apt-get install qemu`

```
#安装 qemu-user
$sudo apt-get install qemu qemu-user qemu-user-static

#此时可以运行静态链接的arm程序，而要运行动态链接的arm程序，需要安装对应架构的动态链接库：
$ apt search "libc6-" | grep "arm"

#安装 qemu-system
$ sudo apt-get install qemu qemu-user-static qemu-system uml-utilities bridge-utils
```

- 源代码安装

要下载并构建QEMU 5.1.0：

```
wget https://download.qemu.org/qemu-5.1.0.tar.xz
tar xvJf qemu-5.1.0.tar.xz
cd qemu-5.1.0
./configure
make
```

要从git下载并构建QEMU：

```
git clone https://git.qemu.org/git/qemu.git
cd qemu
git submodule init
git submodule update --recursive
./configure
make
```

# 异常

但是下载高版本可能会引发异常

异常1.

```shell
onlylove@ubuntu:~/My/qemu/qemu-6.2.0$ ./configure 
Using './build' as the directory for build output
ERROR: GNU make (make) not found
```

解决方法：

```shell
sudo apt-get install make
```

异常2.

```shell
onlylove@ubuntu:~/My/qemu/qemu-6.2.0$ ./configure > log.txt

ERROR: Cannot find Ninja
```

解决方法：

```shell
sudo apt-get install ninja-build
```

# 用户模式使用

> 为了启动Linux进程，QEMU需要进程可执行文件本身以及它使用的动态库。

```
即用户模式要下载运行文件对应的链接库
```

例如，要运行test文件，报错显示缺少/lib/ld-linux-amrhf这个文件

![在这里插入图片描述](https://s2.loli.net/2022/04/18/N6l82q5entf3pvz.png)

**解决方法**

### **1.直接在网上安装对应的共享库**

![在这里插入图片描述](https://s2.loli.net/2022/04/18/IO6HKSYAjwxJobd.png)

这时，所对应的库，就放在如下目录

![在这里插入图片描述](https://s2.loli.net/2022/04/18/8UcH2EBVL3eOiRr.png)

备注：对于arm，我们下载的是 libc6-armhf-cross，其他架构就是替换中间的 armhf 即可。

### 2.将目标二进制所在的源文件系统中的库，拷贝到本地 ubuntu：

将树莓派上的该文件 /lib/ld-linux-armhf.so.3 拷贝到 ubuntu 中，同时也要将相应的库拷贝上，/lib/arm-linux-gnueabihf/。目录结构需要与树莓派保持一致。有以下三种方式，运行动态链接的二进制

对于IoT固件，如果不清楚lib库放在哪，可以直接将整个文件系统复制下来。

#### 1 通过指定环境变量的方式，设置共享库

设置环境变量，要让当前环境默认的根目录更改

```bash
export QEMU_LD_PREFIX=/home/lys/Documents/test
```

这里假设，你将共享库或者整个固件的文件系统放在 /home/lys/Documents/test 目录下（实际上只需要依赖的链接器和库文件），运行成功！

![在这里插入图片描述](https://s2.loli.net/2022/04/18/FMnE4rTYC6LjfZJ.png)

####  2 通过 qemu 参数 -L 设置共享库

```bash
qemu-arm -L /home/lys/Documents/test 目标二进制
```

运行结果如下（假设依赖文件与要执行的二进制在同一目录下，当然可以是不在一个文件夹内的）
 ![在这里插入图片描述](https://s2.loli.net/2022/04/18/13PE6RIian8SMUt.png)

#### 3 通过 chroot 命令，更改当前系统根目录

chroot 命令用来在指定的根目录下运行指令。chroot，即 change root directory （更改 root 目录）。在 linux 系统中，系统默认的目录结构都是以/，即是以根 (root) 开始的。而在使用 chroot  之后，系统的目录结构将以指定的位置作为/位置。因此，为了避免每次都要更改环境变量来运行不同架构的程序，可以使用 chroot  命令，在复制过来的文件系统的根目录下，直接运行。

### 报错

![image-20220418094812470](https://s2.loli.net/2022/04/18/ta3W2dEAChlzeQi.png)

但是实际运行过程中还是遇到了一些小问题，搞了两天还是没弄好环境，等以后再有机会回来看看

