---
title: can协议
date: 2022-05-17 10:12:00
tags:
    - 协议
---

<!--more-->

# CAN

## 基本概念

​		**CAN（Controller Area Network）总线协议**是由 BOSCH 发明的一种**基于消息广播模式的串行通信总线**，它起初用于实现汽车内ECU之间可靠的通信，后因其简单实用可靠等特点，而广泛应用于工业自动化、船舶、医疗等其它领域。

要了解CAN总线协议，我们首先知道的一个概念是汽车ECU是指的什么？ECU（Electronic Control Unit）电子控制器单元，又称为汽车的“行车电脑”，它们的用途就是控制汽车的行驶状态以及实现其各种功能。主要是利用各种传感器、总线的数据采集与交换，来判断车辆状态以及司机的意图并通过执行器来操控汽车。

​		**简而言之CAN总线是用于各个ECU之间互相通信的网络以及协议。**


![img](https://s2.loli.net/2022/05/17/LBCTqeOEslMj1HW.png)

## CAN协议详解

### CAN协议标准

​		CAN总线协议大的分类包含**底层的标准协议**和**上层协议**两种；其中以ISO 11898-1；ISO 11898-2和ISO11898-3这三种协议为主，下面介绍这三种协议的主要作用和应用方向。

- ISO 11898-1: 2015 定义CAN总线的数据链路层（DLL）和电气信号标准，描述CAN总线的基本架构，定义不同CAN总线设备在数据链路层通信方式，详细说明逻辑链接控制（LLC）和介质访问控制（MAC）子层部分；
- ISO 11898-2: 2003 定义高速CAN总线（HS-CAN）物理层标准，最高数据传输速率 1Mbps ，应用为两线平衡式信号（CAN_H, CAN_L），HS CAN是汽车动力和工业控制网络中应用最为广泛的物理层协议；

- ISO 11898-3: 2006 定义低速CAN总线（LS-CAN, Fault-Tolerant CAN）物理层标准，数据传输速率在 40Kbps ~ 125Kbps 。Fault-Tolerant是指总线上一根传输信号失效时，依靠另外的单根信号也可以通信，LS CAN主要应用于汽车车身电控单元之间通信；
  

![img](https://s2.loli.net/2022/05/17/zgvbmJKrUDYTtVo.png)

然而上层协议就更加丰富了，几乎每个厂商都有自己的can总线协议。下图就列举了一些CAN总线的常用上层协议类型：

![img](https://s2.loli.net/2022/05/17/pUQZAjk6xRPbe85.png)

### CAN总线特性

​		CAN总线具有多种特点其中包括：多主的工作方式；每条协议具有不同的优先级；采用**非破坏性总线仲裁技术**；CAN可以通过报文实现点对点、一点对多点以及全局广播方式传送数据；节点数取决于总线驱动电路；采用短帧结构（8/16字节），传输时间短，鲁棒性强，抗干扰；CRC帧校验，数据出错率低。

​		这其中**最重要的特点是多主的工作方式**，一般操作系统都有一个大脑，对整个操作系统的环境进行管理，但是CAN总线的是多主的工作方式，各个ECU只负责往总线上收发它们的协议帧即可，所以当多个ECU同时收发消息时，就会导致冲突，这就又和它第二三个特点相关了。仲裁的特点是基于协议的优先级进行仲裁的，主要是为了给CAN总线上的协议进行优先级排序，决定发生冲突的时候哪个协议先占用CAN总线进行通讯。同时CAN协议的一些特点比如短帧结构，鲁棒性强，抗干扰等等能力也让CAN总线具有了在汽车上适用的条件。


### CAN总线的布局

​		之前汽车的各个ECU之间是通过点对点连接的，但是随着现代汽车内的ECU单元愈发增多，CAN总线连接的方式可以显著降低汽车内部布线的复杂程度。

![图片](https://s2.loli.net/2022/05/17/oLS7RZnbx1Dqze8.png)

​		现代汽车各个ECU的总数加起来可能超过70个如传动控制、安全气囊、ABS等装置；如果考虑到未来无人驾驶汽车的传感器装置，ECU的数量将会达到120个。CAN总线实现汽车内互连系统由传统的点对点互连向总线式系统的进化，大大降低汽车内电子系统布线的复杂度。当然从安全角度出发，总线的布局也更容易导致汽车安全问题。

​		如下图所示，汽车CAN总线的控制布局大致如下图所示，主要包括四个部分：**动力CAN部分，车身CAN，组合仪表CAN，诊断CAN**。动力CAN部分总线主要控制汽车动力以及制动装置部分，这部分对时效性要求比较高，所以一般会使用高速CAN。车身CAN一般是控制空调，车窗，后视镜等等，对时效性要求比较低，一般使用低速CAN。组合仪表部分则是控制汽车组合仪表的CAN。另外还有对汽车接口进行诊断的CAN。

![图片](https://s2.loli.net/2022/05/17/DZcBujKWkEgR1a4.png)

​		**高速CAN（按BOSCH说法，也叫CAN-C）**，数据速率在 125kbps ~ 1Mbp；低速CAN（CAN-B），数据速率在 5kbps ~ 125kbps。高速CAN用在速率比较高的总线上，低速CAN则在实时性要求低的节点，主要在舒适和娱乐领域。这些节点对实时性要求不高，而且分布较为分散，线缆较易受到损坏，低速CAN的传输速度即可满足要求。CAN总线在汽车诊断领域使用的也非常多，汽车诊断领域ECU挂载在总线上。

### CAN总线结构特征

CAN总线定义四种帧类型，分别为**数据帧、远程帧、错误帧和过载帧**。各种帧的用途分别为：

​		**（1）数据帧**：用于发送单元向接收单元传送数据的帧；总线上传输用户数据的帧，其最高有效载荷是 8 Byte，除了有效载荷外，数据帧还包括必要的帧头帧位部分以执行CAN标准通信，比如消息标识符（Identifier）、数据长度代码、校验信息等。


![图片](https://s2.loli.net/2022/05/17/X2ZfWr7Q3TAGsYK.png)

​		数据帧的帧结构如图所示，图中示例标准数据帧（Standard）和扩展数据帧（Extended）两种格式。各字段定义及长度分别为：

```
SOF：表示数据帧开始；（1 bit）

Identifier：标准格式11 bit，扩展格式29 bit包括Base Identifier（11 bit）和Extended Identifier（18 bit），该区段标识数据帧的优先级，数值越小，优先级越高；

RTR：远程传输请求位，0时表示为数据帧，1表示为远程帧，也就是说RTR=1时，消息帧的Data Field为空；（1 bit）

IDE：标识符扩展位，0时表示为标准格式，1表示为扩展格式；（1 bit）

DLC：数据长度代码，0~8表示数据长度为0~8Byte；（4 bit）

Data Field：数据域；（0~8 Byte）

CRC Sequence：校验域，校验算法G(x) = x15 + x14 + x10 + x8 + x7 + x4 + x3 + 1；（15 bit）

DEL：校验域和应答域的隐性界定符；（1 bit）

ACK：应答，确认数据是否正常接收，所谓正常接收是指不含填充错误、格式错误、 CRC 错误。发送节点将此位为1，接收节点正常接收数据后将此位置为0；（1 bit）

SRR：替代远程请求位，在扩展格式中占位用，必须为1；（1 bit）

EOF：连续7个隐性位（1）表示帧结束；（7 bit）

ITM：帧间空间，Intermission (ITM)，又称Interframe Space (IFS)，连续3个隐性位，但它不属于数据帧。帧间空间是用于将数据帧和远程帧与前面的帧分离开来的帧。数据帧和远程帧可通过插入帧间空间将本帧与前面的任何帧（数据帧、遥控帧、错误帧、过载帧）分开。过载帧和错误帧前不能插入帧间空间。
```

​		**（2）远程帧**：用于接收单元向具有相同标识符的发送单元请求数据的帧，远程帧用来向总线上其它节点请求数据的帧，它的帧结构与数据帧相似，只不过没有有效载荷部分。数据帧和远程帧有标准格式和扩展格式两种格式。标准格式有 11 位的标识符 ， 扩展格式有 29 位标识符。一般地，数据是由发送单元主动向总线上发送的，但也存在接收单元主动向发送单元请求数据的情况。远程帧的作用就在于此，它是接收单元向发送单元请求发送数据的帧。远程帧与数据帧的帧结构类似，如上图所示。远程帧与数据帧的帧结构区别有两点：1.数据帧的 RTR 值为“0”，远程帧的 RTR 值为“1” （2远程帧没有数据块远程帧的 DLC 块表示请求发送单元发送的数据长度（Byte）。当总线上具有相同标识符的数据帧和远程帧同时发送时，由于数据帧的 RTR 位是显性的，数据帧将在仲裁中赢得总线控制权。

​		**（3）错误帧**：用于当检测出错误时向其它单元通知错误的帧。表示通信出错的帧。错误标志：6-12 个显性/隐性重叠位。主动错误标志（6个显性位）：处于主动错误状态的单元检测出错误时输出的错误标志；被动错误标志（6个隐性位）：处于被动错误状态的单元检测出错误时输出的错误标志。错误界定符：8 个隐性位
![图片](https://s2.loli.net/2022/05/17/la8RIpPJSYvbeQd.png)

​		**（4）过载帧**：用于接收单元通知发送单元它尚未完成接收准备的帧。在两种情况下，节点会发送过载帧：接收单元条件的制约，要求发送节点延缓下一个数据帧或远程帧的传输；

​		帧间空间（Intermission）的 3 bit 内检测到显性位。每个节点最多连续发送两条过载帧。过载帧由过载标志和过载界定符（8 个隐性位）构成。数据帧的帧结构如图所示。

![图片](https://s2.loli.net/2022/05/17/8My32VGshBQDNIe.png)

## 汽车CAN总线的仲裁机制

​		前面提到过，如果多个节点同时往总线上发送消息，总线的使用权是通过消息帧标识符的逐位仲裁机制决定的，在仲裁过程中消息是不会丢失的。这里的不会丢失的意思是指仲裁完成后，获得总线控制权的消息内容没有被仲裁过程篡改，将继续在总线上发送没有传输完的消息。

​		**在CAN总线上，标识符值越小，消息的优先级越高**。标识符全零的消息，由于它将总线电平保持在显性的时间最长，因此优先级最高。

![图片](https://s2.loli.net/2022/05/17/9xDwO5CvgAFHaBe.png)

​		按照“非破坏性逐位”仲裁机制，就可以从ID一直仲裁到CRC段，可是CAN传输标准并不是这样，CAN标准要求，仲裁仅从基本ID第一位开始，到标准帧的IDE位或扩展帧的RTR位结束。这个区域被定义为仲裁场。如图所示

![图片](https://s2.loli.net/2022/05/17/SiOHWeCPD2plG9b.png)

根据仲裁场范围，CAN总线仲裁流程如图所示：

![图片](https://s2.loli.net/2022/05/17/PSEqM3VAlIUjLu6.png)

​		以上就是CAN总线的所有内容的总结。CAN总线及其协议是汽车安全中非常重要的一个组成部分。想要获取对汽车的控制，对CAN总线协议的了解必不可少。当然，现代汽车的攻击面并不只有CAN总线一个部分，同时还包括其他方面。通过对汽车其他组件的漏洞进行挖掘，同样也能对汽车进行攻击。

## 现代汽车攻击面分析

这里粗略的将汽车的攻击方式分为物理接触攻击和远程攻击。

![图片](https://s2.loli.net/2022/05/17/wzSBp6Y1b42d8sn.jpg)

### 物理攻击的方式

​		(1)**访问车载诊断II（OBD-II）操纵CAN来控制各种模块**，达到的效果是可以控制制动以及发动机模块。另外还可以产生虚假的仪表盘数据，改变发动机参数。

​		对OBD端口进行 Dos服务。只需要产生传输错误即可达到攻击效果，破坏CAN网络。**具体的Dos攻击方式如下**：(i)给支持的参数组号(PGN)发送过量的请求消息，使接收方ECU过载；(ii)发送操纵的虚假请求发送(RTS)，并在接收方缓冲区造成溢出；(iii)通过清除发送(CTS)消息保持连接开放，并占用整个网络。

​		(2)另外就是**使用U盘等物理插入的方式，对车内多媒体设备进行攻击。**

![图片](https://s2.loli.net/2022/05/17/3vfQpmYEdChlA7Z.png)

### 远程攻击方式

​		当今的汽车，包含了与被动防盗、胎压监测系统（TPMS）、蓝牙、无线电数据、远程报文处理等系统通信所需的不同类型的无线接口。这些无线接口需要与CAN通信，通常通过网关ECU来保护网络。黑客可以入侵网关ECU并获得隔离CAN的访问权限。

​		研究员通过逆向工程入侵了汽车的TPMS、蓝牙、FM通道和蜂窝网络，并且小偷可以通过CAN报文解锁车门，轻松盗取车辆。另有研究团队提出了可通过恶意自诊断应用对车辆进行远程攻击。如果有人使用恶意应用程序监测/诊断车辆的情况，攻击者就可以远程控制车辆，实现远距离攻击。

​		两位白帽黑客对12个汽车品牌和21辆商用车开展了远程攻击调查，确定了远程攻击面和攻击对每辆车的危害程度。攻击分为三个阶段。**第一阶段是入侵负责无线接口的ECU。第二阶段是注入报文，与安全关键的ECU进行通信。最后一个阶段是修改ECU，使ECU表现出恶意行为。**研究人员表示，虽然汽车中的日益增多的网络物理系统会增加车辆的脆弱性，但由于汽车中拥有很多不同种类的应用，因此，研究人员无法实际验证车辆的脆弱性。此外，2014年，他们还成功地远程入侵了一辆Jeep切诺基，并使该车的发动机失灵。在发动本次攻击后，他们发布了一则公告，指出了机动车在面对远程攻击时的脆弱。

​		2016年，通过商用远程报文处理控制装置，可以成功地控制了一辆雪佛兰科尔维特的刹车和挡风玻璃雨刷。同时该攻击可通过售后设备渗透到CAN的漏洞中，而对于此，汽车整车厂却无能为力。

​		2016年，通过无线和蜂窝接口，多名研究员对特斯拉Model S实现了远程攻击。腾讯的Keen安全实验室发现了宝马汽车的多个攻击面，这表明即使是高端的商用车也会无法完全避免网络攻击。

​		另外，**通过OTA软件进行攻击也是一种方法。**OTA软件是一种低成本、可扩展、可远程更新的软件解决方案。但同时，这也成为其受攻击的一面。黑客可以通过此软件潜入车辆的通信网络。Beek和Samani通过OTA更新实现了勒索软件攻击。

​		和物理攻击面相比，**现代汽车的远程攻击面所面临的问题更严重。**随着汽车连接性的不断增多，无线攻击面的数量也与日俱增。在不久的将来，汽车将配备车对车(V2V)和车对基础设施(V2I)通信，从而组成车载特设网络(VANETs)。VANETs旨在优化交通、避免碰撞。为了实现上述功能，VANETs使用车辆的传感器并进行无线连接。在车联网中，车辆会接收或传输欺骗报文，车内通信网络可能会因此受到干扰，也会受到攻击。


