---
title: CAN
date: 2022-08-26 14:22:14
tags:
    - can

---

<!--more-->

# CAN协议学习

## CAN连接方式

​	不采用**点对点**方式连接节点，通过**总线**的方式连接节点。

点对点：

​	点对点对于节点少的线路来说还是比较方便的，但是随着现在车辆发展，车上连接的节点越来越多，点对点的方式显然显得需要大量的成本

![image-20220729104146446](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729104146446.png)

总线：

​	所有节点都通用一个介质——总线，线数减少带来线数成本、线数重量、布线难度等都大大减低，同时也带来良好的拓展性，新的节点只要挂载到总线上就行，无需对原有的软件或硬件做修改。但是缺点是整个拓扑通用一个网络结构

![image-20220729104222745](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729104222745.png)

## CAN总线速率

​	高速can最高支持1M，低速can最低125kbs

## 数据链路层

​	两种寻址方式

![image-20220729105348734](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729105348734.png)

```
1.点对点（Peer-to-Peer/P2P(1:1)）
  发送的数据包括目标地址和源地址
```

```
2.广播（Broadcast（1：n)）
  可以实现一对多，发送节点只负责把数据发送到总线上去，不指定接收者，任何在总线上的节点都可以接收发送节点发出的数据，但是接不接收由接收者决定。
```

CAN使用的是第二种广播。

### 仲裁

​	采用非破坏性总线仲裁技术

![image-20220729110155612](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729110155612.png)

![image-20220729111612162](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729111612162.png)

​	假设总线有A、B、C三个节点在发送报文同时访问总线，0代表**帧起始**，开始发送。

​	在0后面是长度为11bite的ID长度，如果大家发送的ID数值一样的话，那是区分不了优先级，但一旦有不一样的数值出现，如上图的A发送了1，B、C都是发送0，那就会产生**线与**的机制（即将发送者的发送的报文**与一下（&）**放到总线上），然后总线会再进行一个**回读机制**，当发送方（A）读到总线上的数据与自己发送的数据不一致时，这时候发送节点（A）会默认自己丢失了仲裁，发送节点（A）就停止发送改为接收节点。但是B和C还未决出优先级，所以它们继续发送数据，直到B和C发送不一致的数据，决出优先级（原理与前文一致）

![image-20220729113042561](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729113042561.png)

这时候B优先级最高，当B发送完时，会进入空闲，这时候A和C会进行新的仲裁，但是上图明显C优先级高，C发送完毕后再是A发送。

```
注意的是：
	CAN ID值是唯一的，所以可以确保弄出优先级的。
```

**反正通俗的来说，ID值为：0~2047，ID值越小，优先级越高**

### 数据帧（CAN Data Frame）

#### 标准格式

![image-20220729135414082](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220729135414082.png)

```
Arbitration Field：仲裁字段

Control Field：控制字段

Data Field：数据字段

Check Field:检查字段

ACK Field：应答域
```

![图片](https://s2.loli.net/2022/05/17/X2ZfWr7Q3TAGsYK.png)

```
SOF：表示数据帧开始；（1 bit）

Identifier：标准格式11 bit，扩展格式29 bit包括Base Identifier（11 bit）和Extended Identifier（18 bit），该区段标识数据帧的优先级，数值越小，优先级越高；

RTR：远程传输请求位，0时表示为数据帧，1表示为远程帧，也就是说RTR=1时，消息帧的Data Field为空；（1 bit）

IDE：标识符扩展位，0时表示为标准格式，1表示为扩展格式；（1 bit）

r:一般作为保留位，在can没什么特殊定义，一般输出显性位0

DLC：数据长度代码，可以表示0~15，0~8表示数据长度为0~8Byte，9~15表示数据长度为8Byte；（4 bit）

Data Field：数据域；（0~8 Byte）

CRC Sequence：校验域，校验算法G(x) = x15 + x14 + x10 + x8 + x7 + x4 + x3 + 1；（15 bit）

DEL：校验域和应答域的隐性界定符；（1 bit）

ACK：应答，确认数据是否正常接收，所谓正常接收是指不含填充错误、格式错误、 CRC 错误。发送节点将此位为1，接收节点正常接收数据后将此位置为0；（1 bit）

SRR：替代远程请求位，在扩展格式中占位用，必须为1；（1 bit）

EOF：连续7个隐性位（1）表示帧结束；（7 bit）

ITM：帧间空间，Intermission (ITM)，又称Interframe Space (IFS)，连续3个隐性位，但它不属于数据帧。帧间空间是用于将数据帧和远程帧与前面的帧分离开来的帧。数据帧和远程帧可通过插入帧间空间将本帧与前面的任何帧（数据帧、遥控帧、错误帧、过载帧）分开。过载帧和错误帧前不能插入帧间空间。

```

### 位填充

连续发送5个相同的位，会插入一个相反的位，任何连续超过5个相同的位都是错误的。

![image-20220802100157112](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220802100157112.png)

### 数据保护机制

1.CRC校验

2.位监控

```
发送方： 会有一种类似回读的机制，当检测到自己发送的信号与bus接收的信号不相同时，会报错。
接收方:  位填充的机制，若接收到大于5个相同的信号，则报错。
```

#### 错误帧

错误帧发送原则：当一个错误被检测时，错误帧会在下一个bite立马被发出来。（CRC除外，CRC会现在ACK发一个1给一个否定应答，然后在ACK DEL 之后才会发一个CRC错误）

#### 错误信号

![image-20220802101509318](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220802101509318.png)

一般错误帧分主动错误节点和被动错误节点。

主动错误节点格式：

```
6个‘0’（+最多6个‘0’）+8个‘1’
```

被动错误节点：

```
6个‘1’+8个‘1’
```

#### 主动被动错误判定

![image-20220802104505740](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220802104505740.png)

```
can错误有三种状态：主动错误、被动错误、Bus Off，状态的切换是通过两个寄存器TEC、REC切换的。
主动错误： TEC<=127 & REC<=127
被动错误： TEC>128 || REC>128
Bus Off: TEC>255 （Bus Off状态下是不会参与任何总线通信）在Bus Off状态下后，只能通过软件重启和等待接收128个11位隐性位（1）才能切换到主动错误状态下
```

## CAN FD总线

CAN FD 支持更高的传输速率，采用可变速率传输。

CAN FD相比于CAN

![image-20220802170242265](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220802170242265.png)

数据帧数据长度变为0~64字节

#### CAN FD 报文具体结构

![image-20220802170655388](C:\Users\Yongpei.Chen\AppData\Roaming\Typora\typora-user-images\image-20220802170655388.png)

CAN FD相比于CAN 增加了几个标志位，以及改动了几位标志位。

```
大致参考CAN数据格式，以下只是关于增加或则修改的标志位
因为CAN FD 没有远程帧，所以RTR 改为了RRS保留位，默认为0
FDF：0为传统CAN报文，1为CAN FD报文
BRS：0为CAN FD 传输速率恒定，1为CAN FD变速传输
ESI：错误状态指示位，1为发送节点为被动错误状态，0为发送节点为主动错误状态
```

