---
title: patch&hook实例操作
date: 2022-05-09 16:32:39
tags:
    - iot
---

<!--more-->

# Patch

## 什么patch

不能获得程序源码，只能直接对二进制文件进行修改，这就是所谓的patch

## patch形式

- patch二进制文件（程序或者库）
- 在内存里patch（利用调试器）
- 预加载库替换原库文件中的函数
- triggers（hook然后在运行时patch）
- 工具进行patch

通过**CVE-2018-20057**来学习patch

### 固件下载

上网搜的固件文件都消失了，找了好久找到

```
链接：https://pan.xunlei.com/s/VN1ckCXDrWvH7QYKPslrTSh0A1
提取码：g3y6
```

### 漏洞文件

/squashfs-root-0/bin/boa

### 虚拟运行

![image-20220509202446702](https://s2.loli.net/2022/05/09/vrfaM3AobwWH81i.png)

返回 `Initialize AP MIB failed!`报错

进入IDA查看进入字符串查找 `Initialize AP MIB failed!`可以追溯到

![image-20220509203511413](https://s2.loli.net/2022/05/09/UY5iRv9N7FwJlnh.png)	

追溯上方函数的跳转条件，可以看到在bnez这里有个跳转判断，当v0=0时，程序会自动跳转到`Initialize AP MIB failed！`这条线

![image-20220509203549902](https://s2.loli.net/2022/05/09/F6g4Hsu7XqhxQlz.png)

### 远程patch

首先使用qemu创建一个远程端口10000，使用命令 `chroot  . ./qemu-mips-static -g 10000 ./bin/boa`

![image-20220509203901025](https://s2.loli.net/2022/05/09/4Nze5QjXyUxhnuM.png)

然后IDA使用远程debugger，连接虚拟机ip的10000端口即可成功远程调试

![image-20220509204222530](https://s2.loli.net/2022/05/09/p85A4la1uHF2fGY.png)

开启之前记得下断点

![image-20220509204311529](https://s2.loli.net/2022/05/09/6QmTYA3iafVPljn.png)

直接运行，可以看到断在跳转位置`bnez    $v0, loc_418DF0`处，并且可以看到右处的v0值为0

![image-20220509204505730](https://s2.loli.net/2022/05/09/ekdjuSLZDfNRFo1.png)

然后，当我们patch 尝试将v0改为1时，可以看到程序不再会报错`Initialize AP MIB failed！`，反而走向另一条支线，这就是patch，这里其实有两种做法，可以修改**bnez**为相反的跳转符**bnqz**（下面就是用这种做法patch）

![image-20220509204957622](https://s2.loli.net/2022/05/09/bzPNQmHYSq2kvLh.png)

可以看一下bnez的地址

![image-20220509210458632](https://s2.loli.net/2022/05/09/qAcaPGX1IizdmQ8.png)

并在hex表找到对应的16进制

![image-20220509210543957](https://s2.loli.net/2022/05/09/K9z8P2prRBa4WgZ.png)

mips架构下这个0x4014好像就是bnez,当修改为0x4010便是bnqz

##### 使用010-editor修改文件

![image-20220509211302007](https://s2.loli.net/2022/05/10/7haRGT316zYpMdl.png)

找到对应位置，将14修改为10，保存后，将修改后的文件（boa_patch）放入虚拟机

![image-20220509211556884](https://s2.loli.net/2022/05/10/riXLpKFfYy8wtMP.png)

### 再次运行

![image-20220509211742006](https://s2.loli.net/2022/05/10/wD758BE4n9TXbRH.png)

可以看到确实没有再出现 `Initialize AP MIB failed！`的错误，但是还是返回了一个段数据报错

![image-20220509212429830](https://s2.loli.net/2022/05/10/DfziCt4kRHQSh7Z.png)通过动态调试可以看到这个段错误报错的地方是这个`apmib_get`，然后又发现这东西是动态链接库上的东西，修改动态链接库就需要涉及Hook技术了

# Hook

## 什么是Hook

Hook其实再学习pwn的时候就接触过，它其实就是通过拦截函数调用，消息或者软件之间传递的事件来改变或者增强操作系统，应用程序或其他软件组件的行为，可以形象地比喻成披着羊皮的狼。所以这里需要用hook欺骗操作系统运行我们编写的文件，让真正的动态链接库文件不执行。

### hook文件

![image-20220509213529404](https://s2.loli.net/2022/05/12/D6Y8eUPBgFfQlJW.png)

其实在v0处一直等于0，与上方这个函数有关，apmib_init也是位于动态链接库的一个函数，可以把v0设为0，所以这里我们还要hook这个函数。

**hook.c**

```
#include<stdio.h>
#include<stdlib.h>
#define MIB_IP_ADDR 170
#define MIB_HW_VER 0x250
#define MIB_CAPTCHA 0x2C1
int apmib_init(void)		//复写apmib_init函数
{
		return 1;
}

int fork(void)
{
	return 0;
}


int apmib_get(int code,int *value)	//复写apmib_get函数
{
		switch(code)
		{
				case MIB_HW_VER:
					*value=0xF1;
					break;
				case MIB_IP_ADDR:
					*value=0x7F000001;
					break;
				case MIB_CAPTCHA:
					*value=1;
					break;
		}
		return 0;
}
```

然后通过

```
mips-linux-gnu-gcc -Wall -fPIC -shared hook.c -o hook.so
解释：
-Wall 生成警告信息
-fPIC -shared 用于生成动态链接库so文件
```

生成so文件

![image-20220509215054525](https://s2.loli.net/2022/05/10/254kGVtErwuzXyK.png)

接着运行命令

```
chroot  . ./qemu-mips-static -E LD_PRELOAD="./hook.so" ./bin/boa
解释：
-E LD_PRELOAD 是执行指定动态链接库
```

成功运行，没有报错段错误，说明hook技术成功运行，并且开了一个80端口

![image-20220509215455077](https://s2.loli.net/2022/05/10/c7JPfqWZiteKu3E.png)

尝试连接ip:80，失败

![image-20220509215641271](https://s2.loli.net/2022/05/10/OvaGk2iQTXWP4FC.png)

原因：在/web文件下有个first.asp,这个文件会对部分访问者进行判断拦截，将它修改为：

```
<html>
<head>
</head>
<% getLangInfo("LangPathWizard");%>
<script>
function init()
{
	var ecflag = <% getIndexInfo("enableecflag") %>;
	if(ecflag == 0)
	{
		if(domain == "CN")
		{
			self.location.href="Basic/Wizard_Easy_Welcome.asp?t="+new Date().getTime();
			//self.location.href="Basic/Networkmap.asp?t="+new Date().getTime();
		}
		else if((LangCode != "EN"))
		{
			self.location.href="Basic/Wizard_Easy_Welcome.asp?t="+new Date().getTime();
		}
		else
		{
			self.location.href="Basic/Wizard_Easy_Welcome.asp?="+new Date().getTime();
			//self.location.href="Basic/Wizard_Easy_LangSelect.asp?="+new Date().getTime();
		}
	}
	else
	{
		self.location.href="index.asp";
	}
}
</script>
<body onLoad="init();">
</html>

```

再次登录ip:80，成功！

![image-20220510084947743](https://s2.loli.net/2022/05/10/z1fcx5bGWOi7odZ.png)
