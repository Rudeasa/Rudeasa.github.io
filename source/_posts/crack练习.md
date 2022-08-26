---
title: crack练习
date: 2021-01-05 12:42:41
tags:
   - re

---

简单练习破解小程序

<!--more-->

# crackme——简单破解小程序

### 找思路



首先拿到一个程序

![image-20210105000928376](https://s2.loli.net/2022/03/26/Q4m238X7jEFepYU.png)

随便 试着输入

![image-20210105005357335](https://s2.loli.net/2022/03/26/siI7au3w6bX28NZ.png)

报错了，查壳

![image-20210105001620383](https://s2.loli.net/2022/03/26/lmLfBzCdn2iceYw.png)

无壳，而且是vb程序，因为crack，所以直接拉到od里查看

![image-20210105002337995](https://s2.loli.net/2022/03/26/tpHei3T57MC9aGd.png)

右键查看字符串，双击进入关键字符串地方“You got it!”然后发现上面是个JE跳转

![image-20210105002536675](https://s2.loli.net/2022/03/26/GPfSOwvNyl3Fcis.png)

跟着地址`004025E5`去查看一下发现跳到了“You Get Wrong”，说明如果执行这个je就会失败，又看到了`004025E5`上面有个jmp，jmp是无条件跳转，跟着jmp后面地址`0040263B`去看一下

![](https://s2.loli.net/2022/03/26/GPfSOwvNyl3Fcis.png)

![image-20210105004131422](https://s2.loli.net/2022/03/26/jfC89r5cmIJhnz6.png)

发现直接结束，所以看到这里可以大概清楚了解这个函数了，只要把一开始的je跳转nop掉，然后让程序执行下来运行到jmp位置直接跳转到程序结束，这样就可以破解这个程序了。

### 具体破解

![image-20210105004505665](https://s2.loli.net/2022/03/26/cwPoCqfO3emuniL.png)

找到

je跳转地方，按空格编译汇编，直接改成nop，再选中修改的两行代码右键复制到可执行文件，点击所有修改，在选择全部复制

![image-20210105004722572](https://s2.loli.net/2022/03/26/SkAGYBEVTqNwxZO.png)

选中修改部分，右击备份保存数据就行。

![image-20210105005158687](https://s2.loli.net/2022/03/26/NpDQgE3WxMI4Uif.png)

最后破解成功

![image-20210105005320373](https://s2.loli.net/2022/03/26/zYNVmT365Oyranw.png)