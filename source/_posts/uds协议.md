---
title: uds协议
date: 2022-05-19 14:13:15
tags:
    - 协议
    - 车联网
---

<!--more-->

# Uds协议

## 概述

UDS定义的服务从逻辑上分为6类，"诊断和通信管理类”、“数据传输类”、“存储数据传输“、“IO控制服务”、“例行程序服务”、“上传与下载服务”。

结合ISO 15765-3和ISO 14229-1则实现了基于CAN总线的UDS汽车统一诊断服务。如下图所示：

![img](https://s2.loli.net/2022/05/19/e6gDPy9sZn4TFiM.jpg)

ISO15765-4规定排放相关的诊断内容

| OSI各层    | 汽车制造商的增强诊断    | 法规要求的排放相关诊断(OBD) |
| ---------- | ----------------------- | --------------------------- |
| 诊断应用   | 用户定义                | ISO 15031-5                 |
| 应用层     | ISO 15765-3/ISO 14229-1 | ISO 15031-5                 |
| 表示层     | 无                      | 无                          |
| 会话层     | ISO 15765-3             | ISO 15765-4                 |
| 传输层     | 无                      | 无                          |
| 网络层     | ISO 15765-2             | ISO 15765-4                 |
| 数据链路层 | ISO 11898-1             | ISO 15765-4                 |
| 物理层     | 用户定义                | ISO 15765-4                 |

UDS不是法规要求的，没有统一实现标准，其优势在于方便生产线检测设备的开发，同时更大的方便了售后维修保养和车联网的功能实现。

诊断通信的过程从用户角度来看非常容易理解，诊断仪发送诊断请求(request)，ECU给出诊断响应（response），而UDS就是为不同的诊断功能的request和response定义了统一的内容和格式。 

### 诊断服务交互方式

UDS本质上是一种定向的通信，是一种交互协议（Request/Response），即诊断方给ECU发送指定的请求数据（Request），ECU反馈给诊断方信息（Response）。

Diagnostic Request的格式：

**Diagnostic Request的格式可以分为两类：**

​    一类是拥有sub-function的；

​    一类是没有sub-function的；

​	长度和格式并没有统一规格，它用于限定诊断服务执行的条件，比如某个诊断服务执行的时间等。Parameter的一个重要应用是作为标识符，标识诊断请求要读出的数据内容。 

有一点要补充的是，其实Sub-function**严格来说是7个bit，而不是1个byte**，因为它的**最高位bit被用于抑制正响应**（Suppress Ppositive Response,  SPR），如果这个bit被置1，则ECU不会给出正响应（PositiveResponse）；如果这个bit被置0，则ECU会给出正响应。这样做的目的是可以告诉ECU不要发不必要的Response，从而节约通信资源。

#### **Diagnostic Response的格式**：

Diagnostic response分为**positive**和**negative**两类。

positive response意味着诊断仪发过来的诊断请求被执行了，而negative response则意味着ECU因为某种原因无法执行诊断仪发过来的诊断请求，而无法执行的原因则存在于negative response报文中的NRC中

Positive response肯定响应的格式是由三部分组成，**其中的response  SID这个字节作为诊断请求的echo，回复[SID+0x40]**，例如请求10，响应50；请求22，响应62，回复的是一组数据。后面的两个字节分别为子功能及其他，视具体的诊断服务而定。

**Negative  response否定响应的格式固定为3个字节，回复[7F+SID+NRC]**，第一个字节为0x7F，第二个字节是被拒绝掉的SID，第三个字节是NRC否定响应码。比如，若ECU给出7F 22 13这个negative  response，则说明SID=22这个服务因为诊断请求数据长度不对（NRC=13）的原因而无法执行。NRC否定响应码如下图所示：

![img](https://s2.loli.net/2022/05/19/IVRzolhJiG1xZrf.jpg)

![img](https://s2.loli.net/2022/05/19/YnkeAiLSFmzEqgB.jpg)

![img](https://s2.loli.net/2022/05/19/k7IMzagtxFi34Ce.jpg)

![img](https://s2.loli.net/2022/05/19/9m43qpu1eriQjFI.jpg)

![img](https://s2.loli.net/2022/05/19/abGcs9WmBCMEY4S.jpg)

![img](https://s2.loli.net/2022/05/19/bMaldHQnNVBqZhC.jpg)

![img](https://s2.loli.net/2022/05/19/y9cVNmBb51Pp8fY.jpg)

举例来说：

​    以CAN总线网络举例。八个帧数据字节，第一字节被网络层占用（ISO 15765-2）。

诊断端发送请求（Request）:02 10 02 xx xx xx xx xx ; 02是网络层单帧SF，代表数据域有2个字节，10是SID，02是子功能。

若ECU的肯定响应（Positive Response）:02 50 02 xx xx xxxx xx；02同上，10+40表示对SID的肯定回复，02是子功能（在下文进行详解）。

若ECU的否定响应（Negative Response）:03 7F 10 22 xx xxxx xx；03同上，代表数据域有3个字节，7F表示否定响应，10是SID，22是NRC。

通俗来说，诊断通信的过程就是诊断仪和ECU交换数据，前者发的是request，后者发的是response，而UDS最重要的作用就是定义了这些request和response的格式和内容。

### 诊断服务内容

UDS协议共定义以下26种服务：

![img](https://s2.loli.net/2022/05/19/m8NLHfT579xZloi.jpg)

![img](https://s2.loli.net/2022/05/19/1342oXJdpBveZzf.png)

## 诊断和通信管理类服务

#### Diagnostic Session Control(0x10)

​    $10用于控制ECU在不同的session之间进行转换，session可以看作是ECU所处的一种软件状态，在不同的session中诊断服务执行的权限不同。其Request请求格式如下所示：

![img](https://pic1.zhimg.com/80/v2-3aea557bba2eae425e9126f2ec253f88_1440w.jpg)

Diagnostic Session Control诊断request的格式

 第一个字节即SID，这里为0x10。第二个自己即$10服务的子功能，功能列表如下所示：

![img](https://s2.loli.net/2022/05/19/DO4b9vczJ7eSC65.jpg)

![img](https://s2.loli.net/2022/05/19/LgBd4SxeD8Gs9lu.jpg)

ECU上电时，进入的是**默认会话0x01（Default）**。如果您进入了一个非默认会话的状态，一个定时器会运转，如果一段时间内没有请求，那么到时间后，诊断退回到默认会话01。当然，我们有一个**$3E的服务，可以使诊断保持在非默认的状态。**

ECU上电之后，由于**默认处在0x01 defaultSession**中，而在这个session中很多诊断服务不可以执行，导致很多诊断相关的数据不能读取或写入。

一般的诊断仪启动之后，会给ECU发送10 03，即让ECU进入 **extended Diagnostic Session**中，在这个session中可执行的诊断服务就很多了。

而如果要让ECU保持在non-defaultSession中，则需要诊断仪每隔固定的时间发送0x3E服务，ECU才会知道诊断仪有和自己通信的需求，从而保持在non-defaultSession中。

另一个常用的session是**0x02 ProgrammingSession**，在这个session中可以进行软件刷写的一系列诊断服务。

**0x40 – 0x5F  这个范围中的session由整车厂自定义使用**，比如，某些诊断服务或诊断数据的操作需要在生产线上执行，即所谓的End-Of-Line，整车厂可以从这个范围中选择一个值来表示EOL session；又或者在开发阶段需要某种“超级”session，则也可以从这里选一个值用来使ECU进入开发模式的session。

DiagnosticSessionControl这个服务非常简单，但是它却是ECU和诊断通信的第一条诊断命令。

####  ECUReset (0x11)

ECUReset 这条指令的用途是通过诊断请求使ECU重启。

![img](https://s2.loli.net/2022/05/19/HqFyifpCKLU73WD.png)

ECUReset 诊断request的格式

第一个字节是SID，即0x11。第二个字节的低7bit是sub-function，用于指示ECU将模拟哪种方式进行重启。 常用的sub-function包括（只举2个例子，UDS还定义了很多其他的值）

0x01 hardReset 模拟KL30的重启

0x02 keyOffOnReset 模拟KL15的重启

当我们通过诊断命令改写了ECU的某些数据，或者对ECU进行了某些设置，只有将ECU重启才能将这些配置生效，所以就有了这个诊断命令。在ECUReset 执行之后，ECU会从Non-defaultsession回退到defaultsession中。

####   SecurityAccess(0x27)

厂家可能会为ECU定义某些安全级别稍微高一些的诊断服务，在执行此类服务之前，就需要执行SecurityAccess 这个诊断命令，进行一个简单的身份验证。

​    完成SecurityAccess 有以下步骤：

![img](https://s2.loli.net/2022/05/19/QxoNAB1anWJjDhM.jpg)

根据UDS对$27服务的定义：

0x03, 0x05, 0x07– 0x41 这个范围留给用于requestSeed的sub-function；

0x04, 0x06, 0x08– 0x42 这个范围留给用于sendKey的sub-function。

具体选择哪对值，由整车厂自己定义。整车厂也可以选择多对sub-function，用于不同等级的安全访问。

举例来说：

​    假设0x05用于requestSeed，0x06用于sendKey。 

1. 诊断仪发送 27 05；
2. ECU响应 67 05 01 0101（seed是 01 01 01）;
3. 诊断仪发送 27 06 02 03 04（key值是02 03 04，seed是 01 01 01，假设本地密码为01 02 03，而算法就是将密码与seed相加）;
4. ECU验证成功 67 06;

​    此时ECU就处于unlocked的状态了，那些被保护起来的诊断服务和诊断数据可以被操作了。通常来说，如果ECU重启，或者回到了default session，unlocked状态就失效了，如果要执行相关诊断服务，则需要再次执行上面描述的过程。

#### CommunicationControl (0x28)

该服务用于打开/关闭某些类别的报文的发送/接收。它通常在刷写软件或大量数据的时候使用，因为在刷软件或参数的时候并不需要ECU进行与通信相关的功能，将通信关闭之后可以把所有通信资源都留给软件或参数的下载，当下载过程完成之后再利用该服务将通信恢复即可。

​    0x28服务的格式如下图所示

![img](https://pic1.zhimg.com/80/v2-9dae508d3412603730636557b7bed814_1440w.jpg)

​    第一部分即SID，一个字节，值为0x28；

​    第二部分是sub-function，即对ECU的通信进行哪种控制，具体包括 ：

![img](https://s2.loli.net/2022/05/19/6I4DnqTPlwg2ek1.jpg)

第三部分表明这条诊断请求要对哪种报文进行控制，长度为1个字节，定义如下表所示：

![img](https://s2.loli.net/2022/05/19/iQBGPzrtlkHnURs.jpg)

​    这个字节中最常用的就是低2 bit，0x1代表普通应用报文，0x2代表网络管理报文，0x3代表普通应用报文和网络管理报文。

​    第四部分是optional的，只有当sub-functional等于0x04或0x05时才需要使用。

举例来说：

​    28 01 01 表示激活应用报文的接收并关闭应用报文的发送（网络管理报文不受影响）。

​    28 00 01 表示激活应用报文的接收和发送（网络管理报文不受影响）。

#### TesterPresent (0x3E)

这个诊断服务的用处可以通过它的名字很明显地得知，即告知ECU诊断仪还在连接着。在$10中提到了关于session的部分，如果没有诊断命令的发送和接收，ECU将从non-default session中回退到default session， 0x3E就是用于使ECU保持在当前session。

这应该是UDS中最简单的一个诊断服务了，它永远只有两个byte，格式如下：

![img](https://s2.loli.net/2022/05/19/w4jEPZrfsitDM3J.png)

当sub-function是0x00时，ECU要给出response；

当sub-function是0x80时，ECU不需要给出response。

一般来说主机厂会为这个服务定义两个时间参数，一个参数用于规定自己的诊断仪发送0x3E服务的间隔，另一个参数用于定义ECU收不到0x3E服务的timeout时间。

#### ControlDTCSetting (0x85)

该服务用于控制ECU的DTC（diagnostic trouble  code，故障诊断码）存储，这个服务常常和前面提到的28服务一起使用，比如，在开始写参数之前，为了获得更快的传输速度，我们用28服务把所有ECU的通信关闭了，但此时因为收到不到相关的报文，ECU会没有必要地存储很多DTC，这时如果我们使用85服务把ECU存储DTC的功能暂时性地禁止掉，则不会造成这种麻烦。

![img](https://s2.loli.net/2022/05/19/uyvNGTmYtzPgfs4.png)

0x85服务请求的格式

​    第一部分即SID，一个byte，值为0x85；

第二部分是sub-function，即是打开还是关闭ECU的DTC存储，包括 ：

0x01 on

0x02 off

​    第三部分是optional的，由各家自己定义，比如，可以用FF FF FF 来表示这条诊断命令针对所有的DTC。

#### ResponseOnEvent (0x86)

尽管诊断通信过程是问答式的，诊断仪发请求，ECU给响应。0x86服务算是一个例外，在ECU收到这条0x86服务之后，当DTC产生时，它会自动地上报DTC及相关环境数据，直到用另一条0x86服务来关闭ECU的这个行为。

该功能主要用于ECU的前期开发阶段，在售后和生产中是不会用到的，而且该服务的格式复杂（即可变的参数很多），执行它还分为好几个步骤，在这里就不过多叙述。

#### LinkControl (0x87)

这个服务用于转化ECU数据链路层和物理层的状态，比如，在高速CAN上的ECU正常通信速率是500kbit/s，但它同时也支持1M bit/s的波特率，如果需要刷写大量数据，便可以利用这条诊断服务让ECU以1M bit/s的波特率进行通信。

这个诊断服务的执行分为两个步骤：

验证ECU是否支持要调整到的目标波特率

让ECU的数据链路层和物理层转到目标波特率的通信状态

只有当第一个步骤验证通过了，第二个步骤才可以成功执行。

## 数据传输类服务

​    在数据传输类服务，0x22和0x2E成对使用，0x23和0x3D成对使用，这几个服务用于诊断数据的基本读写操作。0x24，0x2A，0x2C是一些特殊操作。

#### ReadDataByIdentifier (0x22)

​    $22，即读数据

l Request（请求）：22+DID（Data Identifier，通常是两个字节）

l Response（响应）：62+DID+Data

DID有一部分已经被ISO 14229-1规定了。比如0xF186就是当前诊断会话数据标识符，0xF187就是车厂零件号数据标识符，0xF188就是车厂ECU软件号码数据ID，0xF189就是车厂ECU软件版本号数据标识符。

举例来说：

22 F1 87 （读取零件号，DID=F1 87）

62 F1 87 XX YYZZ KK MM NN（给出零件号）

![img](https://s2.loli.net/2022/05/19/FAKu3sytSBJxlEn.jpg)

![img](https://s2.loli.net/2022/05/19/Iq1tUxTaG9VyCAX.jpg)

![img](https://s2.loli.net/2022/05/19/3qCZXGUPENl9jAb.jpg)

![img](https://s2.loli.net/2022/05/19/4KevE9P3WbaHQxu.jpg)

#### WriteDataByIdentifier (0x2E)

l Request（请求）：2E+DID+Data

l Response（响应）：6E+DID

注意，比如0xF186这个DID不支持直接写入数据，需要用$10来进行会话转换。也就是说，对于写数据的请求，一般来说需要在一个非默认会话，和解锁的状态下才能进行。

举例来说：

2E F1 87 XX YY ZZ KK MM NN（写入零件号）

6E F1 87（给出positive response）

ReadMemoryByAddress (0x23)

![img](https://s2.loli.net/2022/05/19/uVRYsDbQgCGz3hw.jpg)

#### 0x23服务的请求格式

​    第一部分固定为1个byte, 0x23；

​    第二部分是格式信息，长度为1个byte，高4 bits用于指示memorySize的长度（字节数），低4  bits用于指示memoryAddress的长度（字节数）。比如，如果这个值为0x46，则后面的memorySize为6个byte，memoryAddress为4个byte。

  第三部分是memoryAddress信息，它的长度由第二部分的AddressAndLengthFormatIdentifier指示。

  第四部分是memorySize信息，它的长度由第二部分的AddressAndLengthFormatIdentifier指示。

​    如果这条命令的格式是 23 22 xx yy aa bb，则它的含义就是，读取xx yy地址的长度为aa bb的数据。

#### WriteMemoryByAddress (0x3D)

​    了解了0x23的用法，0x3D的用法就很好理解了，它标识memoryAddress和memorySize的方法与0x23相同，只是在诊断命令最后再加上一段需要写入的数据。

## 存储数据传输服务

 存储数据传输服务，用于操作DTC（diagnostic trouble code，故障诊断码）

在 ISO15031-6 中对 DTC 的格式有明确的定义，该规范中定义了 DTC 共由三个字节组成，如下图所示

 DTC格式

| 字节1              | 字节2              | 字节3                    |
| ------------------ | ------------------ | ------------------------ |
| 诊断故障代码高字节 | 诊断故障代码低字节 | 诊断故障代码失效类型字节 |

在 ISO15031-6 中对 DTC 格式中的字节 1 与字节 2 也做了具体的定义，通过这个定义可以很方便确定记录的 DTC 属于车上的那种类型。

![img](https://s2.loli.net/2022/05/19/4pn7CxNhm61RXDU.jpg)

至此，DTC故障码的概念解释完毕，下面将展开关于存储传输类服务的解读：

这类服务主要涉及到两条诊断命令，分别是：

​    0x14:ClearDiagnosticInformation

​    0x19:ReadDTCInformation

​    这两条服务用于操作存储在ECU中的DTC，使用频率很高，而且它们比较好地体现了“诊断”两个字的含义。

#### ClearDiagnosticInformation（0x14）

这条诊断命令的格式比较简单，用法也很好理解，即删除存储在ECU中的DTC。

第一个字节就是SID了，后边的三个字节用于标识将要被删除的DTC种类，UDS规定用FF FF FF表示所有种类的DTC，由厂家自定义代表Powertrain、Chassis、Body、NetworkCommunication等种类DTC的值。

举例来说

​            Request：14+FF+FF+FF；（删除掉ECU中的所有DTC）

​            Response：54 ；

![img](https://s2.loli.net/2022/05/19/YKw913cZyrsFQhX.png)

#### ReadDTCInformation（0x19）

这条指令用于读取存储在ECU中的DTC，0x19服务的sub-function代表了各式各样读取DTC的方法，UDS给19服务的sub-function从0x00到0x19进行了明确定义，这里介绍其中常见4种。

- sub-function = 0x01（reportNumberOfDTCByStatusMask）

用于读取符合DTC状态掩码的DTC的数量，此时parameter为一个byte的Mask掩码，用于与DTC的Status进行“与”运算，而ECU返回的则是"与"运算之后结果不为0的DTC的数量。

DTC的Status用一个byte表示，其中的8个bit分别代表DTC的不同状态，比如，bit0 表示这个DTC是active的还是passive的，bit  4表示这个DTC是否已经被confirm了，如果DTC的状态是confirm，则说明该DTC已经被ECU存储下来了。

比如：19 01 08这个命令的用途，就是读取所有状态为confirm的DTC的数量。

- sub-function = 0x02（reportDTCByStatusMask）

用于读取符合特定条件的DTC列表，此时parameter仍然为一个byte的Mask，用于与DTC的Status进行“与”运算，而ECU返回的则是"与"运算之后结果不为0的DTC列表。

比如19 02 01这个命令的用途，就是读取所有状态为active的DTC的数量。此时ECU返回的格式应该是59 02 01 XX XX XX 01  YY YY YY 09......。返回的DTC列表中的每个条目为4个字节，前三个字节用于标识DTC，比如 XX XX  XX，最后一个字节用于标识DTC状态，比如01，表示DTC是active的，09表示DTC是active且confirm的。

- sub-function = 0x06（reportDTCExtDataRecordByDTCNumber）

用于读取某个DTC及其相关的环境数据，此时parameter为4个byte，前三个byte用于标识我们要读取的DTC，第四个byte用于标识要读取的环境数据的范围，UDS规定使用FF来表示读取所有的环境数据，各厂家可以要根据自己的需求定义其他的值来代表要读取的环境数据的范围。环境数据包括DTC状态，优先级，发生次数，老化计数器，时间戳，里程等，厂家还可以根据自己的需求定义一些此DTC产生时的测量数据。

比如 19 06 XX XX XX FF就表示读取 XX XXXX这个DTC的所有环境数据，ECU的返回值应该是59 06 XX XX XX AA  BB CC DD.....，其中AA BB CCDD...代表的就是XX XX XX这个DTC产生时所一起存储的环境数据。

- sub-function = 0x0E（reportMostRecentConfirmedDTC）

 sub-function =  0x0E时，不需要parameter。0x0E表示，要求ECU上报最近的一条被置为confirm的DTC。上文介绍过0x86服务，sub-function = 0x0E的19服务通常被作为参数传递给86指令，要求ECU在发生DTC存储的时候进行自动上报，即19  0E这两个字节的指令被嵌入到86服务的命令中。这条命令在开发阶段会用到，比如验证某个故障路径是否生效。

## I/O控制服务

#### InputOutputControlByIdentifier (0x2F)

$2F这个服务即利用ID对ECU的输入输出进行控制，经常会在生产线上使用来验证零部件的功能。比如，在总装阶段，工人需要验证车上的各种功能是否正常，例如四个车窗的升降是否正常，如果挨个开关去按，那效率很低，如果通过一个诊断命令就能够观察到车窗升降的情况，效率则高得多。

比如，ECU接收一个输入信号A，我们就可以利用2F给这个A赋个我们需要的值；ECU对某个执行器B进行控制，我们就可以利用2F服务再配上某些特定的参数来实现对B的控制，例如门控对车窗升降、后视镜折叠等的控制。

$2F服务的request由4部分组成

第一个字节：SID=0x2F

第二三个字节：dataIdentifier(2 byte)，用于标识被控制的IO对象，例如下文举例0x9B00（进气口门位置）

第四个字节：controlOptionRecord，用于标识控制类型，以及若干byte由厂家自定义使用的controlState。UDS明确定义了四种控制类型

​            0x00 将控制权还给ECU，即结束控制

​            0x01 将DID标识的对象的输入信号、内部参数、输出信号等设为默认值

​            0x02 将DID标识的对象的的输入信号、内部参数、输出信号等冻结住

​            0x03 将DID标识的对象的的输入信号、内部参数、输出信号进行设置，其实就相当于开始了对ECU的控制，这种控制类型其后会跟着第五个字节，可以表示控制对象的数字量或模拟量。

举例来说：

​    使用2F控制Air Inlet Door Position （进气口门位置），用DID（标识符）=0x9B00来标识进气口门的位置。

Air Inlet Door Position [%] = decimal(Hex) * 1 [%] ，即用一个百分比来表示这个位置。

**step1:**

tester 发送22 9B 00读取当前进气口门的位置，这里22即SID，0x9B00即第二三字节DID。

ECU返回62 9B 00 0A ，这里 0x0A = 10（dec），即表示当前位置是10%

（UDS定义可以用22服务读取2F服务中使用的dataIdentifier，返回值是状态信息，具体的状态信息是什么，则由使用者自定义了。）

**step2:**

tester 发送2F 9B 00 03 3C ，表示要将进气口门的位置调整到60%，0x3C = 60（dec）表示要将控制对象的模拟量设定为60%。

ECU返回6F 9B 00 03 0C，表示接受控制，当前进气口门的位置为12%。因为ECU收到请求后是立刻响应的，而门的位置调节需要时间，所以还没有达到60%。

**step3:**

过一段时间后tester 发送22 9B 00读取当前进气口门的位置

ECU返回62 9B 00 3C ， 0x3C = 60（dec），表示当前位置已经到了60%

**step4:**

tester 发送2F 9B 00 00，将控制权交还给ECU

ECU返回6F 9B 00 00 3A，表示接受请求，当前位置为58%

**step5:**

tester 发送2F 9B 00 02，冻结9B 00这个ID所代表的进气口门位置这个状态

ECU返回6F 9B 00 02 32，表示接受请求，当前位置保持在50%

**此外，2F服务存在一个问题，如果通过2F服务修改了某个值，后续也不把控制权还给ECU，那么这个修改的有效时间是一直持续下去？正确的做法是如果扩展会话超时，即切回默认会话，此时控制权应还给ECU，毕竟 2F的03子功能是"暂时接管控制权"。**

## 上传与下载服务

  关于ECU升级数据的传输，是通过34（请求下载）、36（传输数据）、37（请求退出传输）等服务来完成的。由于汽车ECU中用于缓存诊断服务数据的buffer大小有限，所以当我们需要读取或写入超过buffer大小的数据时，就无法简单地使用2E和22服务了，因此UDS据此定义了几个将大块数据写入或读出的服务，即数据下载和上传。

Upload Download functional unit总共定义了5个诊断服务，分别是：

**RequestDownload （0x34）**：诊断仪向ECU请求下载数据

**RequestUpload （0x35）**诊断仪向ECU请求上传数据

**TransferData（0x36）** 诊断仪向ECU传数据（下载），或者服务器向客户端传数据（上传）

**RequestTransferExit（0x37）**数据传输完成，请求退出

**RequestFileTransfer（0x38）** 传输文件的操作，可以用于替代上传下载的服务。

下图是数据下载的简略过程，用到了34，36，37这三个服务，如果是上传的话，34服务被35服务替换，数据传输方向变一下，就可以了。

![img](https://s2.loli.net/2022/05/19/WeGlnwymPOpqYAC.jpg)

**step1:**

​    诊断仪通过34服务传输该块的起始地址、该块的数据长度信息；进行下载请求；

​    ECU收到34服务的下载请求后，通过74肯定响应报文通知诊断仪，其（诊断仪）接下来的每个数据传输的报文中（36服务）应包含多少数据字节。诊断仪则根据该返回的参数对自身的发送能力进行调整；

**step2:**

​    诊断仪通过36服务传输该块的数据，每个36服务传输的数据量大小由ECU返回的34服务（即SID=0x74）中的参数确定，详情见下文。

​    ECU对36服务返回肯定响应；

​    通过36服务依次将该数据进行拆分发送，期间每完成一次36服务的发送，ECU进行肯定响应的回复。直到该块数据全部发送完；

**step3:**

​    诊断仪通过发送37服务进行传输退出的请求；

ECU进行肯定响应回复。

各服务说明如下所示：

各服务说明如下所示：

#### **RequestDownload （0x34）：**

0x34服务用于启动下载传输，作用是告知ECU准备接受数据，ECU则通过0x74 response告诉诊断仪自己是否允许传输，以及自己的接受能力是多大。

 0x34服务的请求格式包括5个部分

​    第一部分：1个byte的SID=0x34

​     第二部分：1个byte的dataFormatIdentifier，用于指示数据压缩和加密的方法。其中，第7-4位定义了压缩方法；第3-0位定义了加密方法。该字节为0x00时则表示既没有使用压缩方法也没有使用加密方法。0x00外的其他值的定义是由车产自行规定的。

​     第三部分：1个字节的addressAndLengthFormatIdentifier，用于指示后面两个部分所占用的字节数，高4bit表示memorySize所占的字节长度，低4bit表示memoryAddress所占的字节长度。在这个例子中我将这两个值分别设置为n和m。

 第四部分：m个字节的memoryAddress，由addressAndLengthFormatIdentifier中的低4bit指示。含义是要写入数据在ECU中的逻辑地址。

第五部分：n个字节的memorySize，由addressAndLengthFormatIdentifier中的高4bit指示。含义是要写入数据的字节数。

 ECU收到请求之后，如果允许传输的话，会给出如下response

第一部分：1个byte的 Response SID=0x74

​    第二部分：1个byte的dataFormatIdentifier作为echo

第三部分：maxNumberOfBlockLength，长度不定，表示可以通过0x36服务一次传输的最大数据量。

#### **TransferData（0x36）：**

如果34服务得到了正确响应，tester就要启动数据传输过程了，使用的就是36服务。36服务的格式如下。

 第一部分：1个byte的 SID=0x36。

 第二部分：1个byte的blockSequenceCounter，标识当前传输的是第几个数据块，即帧序号，或者简单地说就是第几次调用36服务。

  第三部分：transferRequestParameterRecord，传输的数据。第次传输数据量的上限就是34服务响应中的maxNumberOfBlockLength。

举例：如果ECU告知tester,maxNumberOfBlockLength =  20（来自$34服务的echo），也就是说tester每次通过36服务只能发送最多20个字节，其中还包括了SID和blockSequenceCounter，所以实际上每次可传的数据信息只有18个字节。如果tester要传的数据为50个字节，则需要传输三次，每次分别传输18，18，14个字节，即调用3次36服务。

36的响应很简单，就是一个字节的Response SID再加一个字节的blockSequenceCounter作为echo。

#### **RequestTransferExit（0x37）：**

 $37服务用于退出上传下载，如果之前的34和36服务都顺利执行完成，那么37服务就可以得到ECU的positive response。

​    格式很简单，请求就是37，正确响应就是77，都是一个字节。

如果前面的36服务没有执行完成，以我前面举的例子来说，比如这个数据块有50个字节，但是tester只发了两次36服务传了36个字节，那么这次传输对于ECU来说是失败的，所以ECU应该给出NRC 0x7F 37 24，表示诊断序列执行有错误。

## UDS应用的设备

在UDS诊断产品中知名度最高，应用最广泛的是德国Vector公司的CAN卡 VN1630/1640 配合其CANoe 软件，Vector  产品功能齐全，适合系统级汽车总线开发，被大部分汽车厂商采用。通常工程师先用Vector的CANdela进行cdd文件的开发，之后将该cdd文件导入CANoe.diva中进行功能测试。下面的链接是Vector提供的全套解决方案，里面的CANdesc是UDS代码生成软件。

 Vector 产品很好用，节省开发时间，不开放API，且价格昂贵，不适用于硬件开发团队和生产线的自动化测试。目前市面上有很多CAN 厂商（如Kvaser, ZLG 等）能提供低成本、体积小、驱动简单、开放API 的设备，很适合进行二次开发。
