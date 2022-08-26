---
title: doip协议
date: 2022-05-18 08:40:03
tags:
    - 协议
---

<!--more-->

# doip协议

DoIP协议（Diagnostic On IP---ISO 13400）定义将IP技术运用到车载网络诊断范畴的通信规则。其中包括两层含义：

1、 将IP技术应用到车载网络中，需满足车规需求；

2、 在诊断范畴，DoIP协议定义了从物理层（Physical Layer）到应用层（Application Layer）搭建“通信桥梁”的规则（此处可类似CAN总线的TP层协议ISO 15765-2）

将上述概念映射到OSI计算机七层模型：

![image-20220518092224083](https://s2.loli.net/2022/05/18/cmGLPxJUrD51AZg.png)

##  数据链路层与物理层

根据ISO-13400的要求，DoIP通信在物理层支持**100BASE-TX (100 Mbit/s Ethernet) 和10BASE-T (10 Mbit/s Ethernet)** 两种制式。

## 传输层与网络层

DoIP设备的MAC地址也符合IEEE 802.3 的要求，doip主要是在传输层与网络层工作。

ISO-13400规定，DoIP通信在传输层上**必须同时支持UDP和TCP**，并将UDP和TCP的使用场合进行了定义，对所使用的端口号也进行了定义。

ISO-13400规定，DoIP通信在网络层上**使用IPv6协议，但是为了后向兼容的原因，也支持IPv4**。此外，对于IPv4来说，还要支持地址解析协议（ARP ），对于IPv6来说，还要支持邻居发现协议(NDP) ，这两个协议是用于在只知道IP地址的情况下获取MAC地址的。

#### **ARP格式包**

![img](https://s2.loli.net/2022/05/18/MdnZcmtGjuysBP4.png)

#### **NDP数据包**

参考 https://blog.csdn.net/zhishenluo/article/details/103729512 

## DoIP数据帧格式

### 帧格式说明

**以太网帧（具体参考网络帧）**

![img](https://s2.loli.net/2022/05/18/ny2OiFfPxNLVIsj.png)

##### **IP段**

![img](https://s2.loli.net/2022/05/18/ivlCf9aAeuZTF6S.png)

##### **TCP段**

![img](https://s2.loli.net/2022/05/18/gcs5EvJq9wjMUkZ.png)

##### **UDP段**

![img](https://s2.loli.net/2022/05/18/2KSv5Hxnu34Pbta.png)

##### **DoIP段**

![img](https://s2.loli.net/2022/05/18/DbNJG8OziToqgBI.png)

### **DoIP-协议版本**

![img](https://s2.loli.net/2022/05/18/EdQMfreP9qZY1g4.png)

```
0x00: reserved     保留

0x01: DoIP ISO/DIS 13400-2:2010

0x02: DoIP ISO 13400-2:2012      

0x03…0xFE: reserved by this part of ISO 13400    由ISO 13400本部分保留

0xFF: default value for vehicle identifcation request messages     车辆识别请求消息的默认值
```

### **DoIP-Data Type**

![img](https://s2.loli.net/2022/05/18/EqlVQ4yeTm8IR7x.png)

![img](https://s2.loli.net/2022/05/18/KT7o4Bu1aPiO6Zp.png)

【0x0001至0x0004】用于汽车标识上报或请求，只能通过UDP报文来发送这种命令，主要用于在汽车和诊断仪进入网络之后**、诊断连接建立之前的车辆发现过程**。

【0x0005 和0x0006】标识的Routing activation request 和 response用于在socket建立之后，**进行诊断通信的请求**。

【0x0007和0x0008】用于Alive check，用于**检查当前建立的诊断连接socket是否仍然在使用中**，如果不再使用，则关闭socket释放资源。
		【**0x8001，0x8002，0x8003】，**分别代表的含义分别是诊断消息、诊断消息	正响应和诊断消息负响应。

### **DoIP-Data length**

![img](https://s2.loli.net/2022/05/18/HcefnzothjK8FNr.png)

![img](https://s2.loli.net/2022/05/18/IShYn4sBjV5AUfo.png)

就是标识后面的user data的长度。

此外源地址和目标地址可以参考UDS中定义即可，用户数据即为诊断相关服务数据。

## 诊断连接

### 连接状态

![img](https://s2.loli.net/2022/05/18/sj1TpeCJ59Q4ghb.png)

DoIP实体内管理着一个DoIP connection table ，用来记录和维护诊断通信的逻辑连接。上图就是这个表中的一个元素，即一个逻辑连接的状态机。上图中的方框就是连接所处的状态，[Step]是状态之间跳转时发生的事情。

[Step1] 当一个新的套接字建立，逻辑连接的状态就从“listen”跳转到“socket initialized”，同时启动一个定时器， initial inactivity timer。

[Step2] 当DoIP实体接收到tester发来的一个routing activation信息后，逻辑连接的状态就从“socket initialized”跳转到“Registered [**Pending for Authentication**]” ，此时 initial inactivity timer被停止，启动一个名为general inactivity timer的定时器。

[Step3] 在完成Authentication之后，逻辑连接的状态就从“Registered [Pending for Authentication]”跳转到“Registered [**Pending for Confrmation**]” 。

[Step4] 在完成Confrmation之后，逻辑连接的状态就从“Registered [Pending for Confrmation]”跳转到“Registered [**Routing Active**] ” 。

[Step5] 如果initial timer 或general inactivity timer 过期后仍没收到后续请求，或者authentication 和 confrmation 被拒绝了，又或者外部测试设备对alive check 消息没有响应，则逻辑连接进入“Finalize”状态。

[Step6]进入Finalize后，此时TCP套接字将被关闭，并重新回到“listen”状态。

### **建立连接和车辆发现**

![img](https://s2.loli.net/2022/05/18/YPRIaDh8JgGjM4f.png)

当DoIP实体和外部测试设备都连接到一个网络中时，它们会利用DHCP协议获得一个属于自己的IP地址。在网络中，路由器作为DHCP server，为新加入到该网络中的设备分配IP地址。在获取IP地址之后，有两种车辆发现的方法，如上图所示。一种方法是车辆主动上报自己的信息3次。如果测试设备没有收到车辆主动上报的信息，则会发送一个identification request，如果网络中有车辆的话，车辆对这个请求进行响应，测试设备便发现了被测车辆。


###  **会话建立**

![img](https://s2.loli.net/2022/05/18/eZasTkjv7DxWzQH.png)

在诊断仪发现车辆之后，会把车辆添加到自己的车辆列表中。当用户选择这个列表中的某辆车，如果连接建立成功，用户就可以对车辆进行诊断了。

接下来用户给汽车发出诊断信息，网关会根据信息接收对象把诊断信息转发给网络中相关的ECU，当得到ECU 的响应之后，网关再把最终的响应发送给诊断仪。当用户选择退出时，用于DoIP通信的这个套接字就被关闭了。


## **诊断发送**

### **请求DID F810读取**

![preview](https://s2.loli.net/2022/05/18/s5QbduRVjmo2JBt.png)

```
byte 0：ISO13400 版本

byte 1：ISO13400 版本逐比特取反

byte 2~3：数据类型，0x8001，表明这是一个诊断信息的数据包

byte 4~7：数据长度，在这个例子中的值是7，表示后面有7个字节的数据

byte 8~9：源地址

byte 12：具体的诊断命令，SID：22，表示读取

byte 13~14：DID-->Data Identifier，通常是两个字节，举例是0xF810

其他诊断服务类似。
```

