---
title: 0day安全软件漏洞学习第四弹
date: 2022-04-07 11:14:44
tags:
    - 漏洞学习
---

<!--more-->

# 文件类型漏洞挖掘 与 Smart Fuzz
​		fuzz工具的大名早就有所耳闻，之前一直想接触接触漏洞挖掘方面，但是不明白fuzz工具的用法，也不知道在什么环境下使用，恰好《0day安全软件漏洞学习》这本书里有介绍，借此机会好好学习整合一下fuzz方面的知识吧。

​		最近看到书的第16章节了，前面几章节介绍的很多基础知识点我都自己已经整理过就不再记录了，看了一下大部分都和Linux中的一些保护机制大同小异（canary、NX、pie……）,因为看着很熟悉，也就很快的过一遍，不过作者介绍的是windows 2000及windows XP系统下的这些漏洞，漏洞原理和Linux中的差不多。

​		不扯废话了，接下来进入主题。

## 什么是FUZZ

​		Fuzz本意是“羽毛、细小的毛发、使模糊、变得模糊”，后来用在软件测试领域，中文一般指“模糊测试”，英文有的叫“Fuzzing”,有的叫“Fuzz Testing”。它是一种软件测试技术，通常是自动或半自动的，它涉及将无效、意外或随机的输入传递给程序，并监控崩溃、失败的断言、竞争、泄漏等结果。（fuzz是一门技术，之前的我一直以为是一种软件专门用来进行漏洞挖掘的工具。）

## 作用

Fuzzing是模糊测试，顾名思义，意味着测试用例是不确定的、模糊的。计算机是精确的科学和技术，测试技术应该也是一样的，有什么的输入，对应什么样的输出，都应该是明确的，那fuzz的模糊测试存在就是为了预防未知的漏洞。

## fuzz产生原因

1.我们无法穷举所有的输入作为测试用例。我们编写测试用例的时候，一般考虑正向测试、反向测试、边界值、超长、超短等一些常见的场景，但是我们是没有办法把所有的输入都遍历进行测试的。

2.我们无法想到所有可能的异常场景，由于人类脑力的限制，我们没有办法想到所有可能的异常组合，尤其是现在的软件越来越多的依赖操作系统、中间件、第三方组件，这些系统里的bug或者组合后形成的bug，是我们某个项目组的开发人员、测试人员无法预知的。

3.Fuzzing软件也同样无法遍历所有的异常场景。随着现在软件越来越复杂，软件无法遍历所有输入，否则你的版本可能就永远也发布不了。Fuzz ing技术本质是依靠随机函数生成随机测试用例来进行测试验证，所以是不确定的。

## 结果

我们只要重复fuzz技术通过随机函数产生的随机测试用例大量地去测试，那些概率极低地的偶然事件类似bug必然就会出现。

## 生成方式

（1）基于变异：根据已知数据样本通过变异的方法生成新的测试用例

（2）基于生成：根据已知的协议或接口规范进行建模，生成测试用例

传统的 fuzz 大多通过对已有的样本 按照预先设置好的规则 进行变异产生测试用例，然后喂给 目标程序同时监控目标程序的运行状态，这也是为什么fuzz适合漏洞挖掘的原因，这类 fuzz 有很多，比如: peach , FileFuzz 等。接下来以书中的FileFuzz为例讲解。

### 文件格式Fuzz

一个File Fuzz工具大体的工作流程包括以下几步。 

（1）以一个正常的文件模板为基础，按照一定规则产生一批畸形文件。 

（2）将畸形文件逐一送入软件进行解析，并监视软件是否会抛出异常。 

（3）记录软件产生的错误信息，如寄存器状态、栈状态等。 

（4）用日志或其他UI形式向测试人员展示异常信息，以进一步鉴定这些错误是否能被利用。 

![image-20220408164934216](https://s2.loli.net/2022/04/08/LSbIfE9XNyF8WjQ.png)

#### Blind Fuzz和Smart Fuzz

​		Blind Fuzz即通常所说的**“盲测”**，就是在随机位置插入随机的数据以生成畸形文件。然而现代软件往往使用非常复杂的私有数据结构，例如PPT、word、excel、mp3、RMVB、PDF、Jpeg，ZIP压缩包，加壳的 PE 文件。数据结构越复杂，解析逻辑越复杂，就越容易出现漏洞。复杂的数据结构通常具备以下特征： 

- 拥有一批预定义的静态数据，如magic，cmd id等 
- 数据结构的内容是可以动态改变的
- 数据结构之间是嵌套的
- 数据中存在多种数据关系 （size of，point to，reference of，CRC）
- 有意义的数据被编码或压缩，甚至用另一种文件格式来存储，这些格式的文件被挖掘出越来越多的漏洞…… 

对于采用复杂数据结构的复杂文件进行漏洞挖掘，传统的Blind Fuzz暴露出一些不足之处，例如：产生测试用例的策略缺少针对性，生成大量无效测试用例，难以发现复杂解析器深层逻辑的漏洞等。

 针对Blind Fuzz的不足，**Smart Fuzz**被越来越多地提出和应用。通常Smart Fuzz包括三方面的特征：面向逻辑（Logic Oriented Fuzzing）、面向数据类型（Data Type Oriented Fuzzing）和基于样本（Sample Based Fuzzing）。 

书中介绍的三方面smart fuzz特性，我觉得用自己话来总结就是

1. 面向逻辑：我们要精准地分析要在测试样例哪一层逻辑试探，相比起盲测，它可以躲过上层的解析器的探查；
2. 面向数据类型测试：识别不同的数据类型（算术型：包括以HEX、ASCII、Unicode、Raw格式存在的各种数值。指针型：包括Null指针、合法/非法的内存指针等。字符串型：包括超长字符串、缺少终止符(0x00)的字符串等。特殊字符：包括#, @, ‘, <, >, /, \, ../ 等。 ），并且能够针对目标数据的类型按照不同规则来生成畸形数据。
3. 基于样本：变异测试需要依靠的是修改样本的参数来实现，并且变异测试只能对样本涵盖的功能测试，如果样本不包含所有功能，那么只能测试样本中的功能，为了提高测试质量，就要求在测试前构造一个能够包含几乎所有数据结构的文件来作为样本。

### peach介绍

在使用Peach进行Fuzz之前需要编写被称为”Peach Pit”的xml配置文件，其中包含着如何进行Fuzz的关键信息

![image-20220410104639393](https://s2.loli.net/2022/04/10/4uYIzfsogO6xRnL.png)

其主要元素包括：

- **DataModel**

- **StateModel**

- **Agent**

- **Test**
- ……



#### DataModel

一个Pit文件至少会包括一个或多个DataModel，描述数据类型信息，关系（大小、数量、偏移量），和其它允许智能Fuzz的信息，如下

![image-20220410105106302](https://s2.loli.net/2022/04/10/XDTqad9CbgE6MGu.png)

其属性包括：

> 1) name：数据模型的名字[必须]。
>
> 2) ref：引用模版数据模型[可选]。DataModel有ref属性时，与被引用DataModel类似

于子类与基类的关系，基类数据会被子类继承，子类子元素会覆盖基类同名子元素，

> ​	3.mutable：数据元素可变异性[可选，默认true]。

mutable主要子元素：Blob、Block、Choice、Flags、String、Number、Relation等。

1) Blob：常用于表示没有类型定义和格式的数据，如下图：

[![image005](https://s2.loli.net/2022/04/10/MQP13zlxvWhguUE.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image005.png)

其主要属性包括：

- value：Blob默认值。
- length ： blob的字节长度， blob长度判断会根据后续有token元素的位置计算。
- token：这个元素解析是否作为”标记”，默认false。

2) Block：用来组合一个或者多个的其他元素。Block和DataModel是很类似的，一个重要区别在于它们的位置，DataModel是顶级元素， Block是其子元素。

其不同于DataModel的属性包括：

- minOccurs：这个Block所必须出现的最低次数[可选] 。
- maxOccurs ：这个Block可能会出现的最高次数[可选]。

3) Choice：每次选择其中一个元素，类似switch语句。

![image006](https://s2.loli.net/2022/04/10/6WjdQlZMsaC1LFf.png)

minOccurs为最小生成Choice数；maxOccurs为最大生成Choice数，-1为无上限；occurs为必须产生的次数，如果不能达到这个次数，异常退出。具体匹配实现按照Choice中Block顺序，crack（解析）数据时根据token匹配一个Block后，数据位置后移匹配Block大小，继续按照Choice中Block顺序从头匹配。

4) Flags： Flag元素定义包含在Flags容器中的位字段，如下图：

![image007](https://s2.loli.net/2022/04/10/vzu3P8QW4CZEbOa.png)

其主要属性包括：

- size：大小，以位数为单位[必须] 。
- position：flag的起始位置（以0为基准）[必须]。

5) String：定义一个或者双字节的字符串，如下图：

[![image008](https://s2.loli.net/2022/04/10/wTMRkl8u4KB9W6A.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image008.png)

其主要属性包括：

- nullTerminated：字符串是以null结尾[可选] 。
- type：字符编码类型，默认”ascii”，可用选项有ascii, utf7, utf8, utf16, utf16be,

utf32 [可选]。

- padCharacter：填充字符串，来填充达到length的长度，默认是0x00[可选]。

6) Number：定义了长度为8，16，24，32 或者64位的二进制数字，如下图：

[![image009](https://s2.loli.net/2022/04/10/hAosjOgVcESmYGv.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image009.png)

其主要属性包括：

- size：Number的大小，以位为单位。有效的选择是1-64 [可选]。
- endian：数字的字节顺序，默认是小端字节［可选］。
- signed：是否是有符号，默认是true[可选]。

7) Relation：用于连接两个大小、数据、偏移量相关元素，如下图：

[![image010](https://s2.loli.net/2022/04/10/MAwKcH7jVmyGguF.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image010.png)

type类型为size时，of表示Number 是Value字符串的字节数。expressionGet用于crack过程，表示读”Value”多少字节。expressionSet用于publishing过程，为Publisher 生成Number值。

#### StateModel

用于定义测试的逻辑，实际上相当于一个状态机。如下图：

![image011](https://s2.loli.net/2022/04/10/VvDNPdXIyOSfeFU.png)	

下级标签包括State，每个State中又可以包含若干个Action标签。

1) State：表示一个状态，不同的State之间可以根据一些判断条件进行跳转，通常和Action的when属性联合使用。如下图：

![image012](https://s2.loli.net/2022/04/10/vHDJFMXYE5bo1iW.png)

\2) Action：用于完成StateModel中的各种操作，是给Publisher发送命令的主要方式。Action能发送输出、接收输入、打开连接，也能改变State等。主要属性：

- type：操作类型[必须]。主要类型：

start：启动Publisher，隐含动作，一般不需要。

stop：停止Publisher，隐含动作，一般不需要。

input：接收或者读取来自Publisher的输入，需要指定DataModel，用于crack和包含输入数据。

ouput：通过Publisher发送或者写输出，需要一个DataModel ，包含可选data，如下图：

![image013](https://s2.loli.net/2022/04/10/towFpnYIDeh5PdX.png)

- when：如果提供的表达式为true，完成操作；否则，跳过。
- ref：状态变更后的引用[type=changeState] 。
- method：call的方法 [必须, type=call]，调用Publisher可选参数定义的方法，不是所有Publisher都支持。

#### Agent

是能够运行在本地或者远程的特殊的peach进程，这些进程能够启动监视器监控被测目标，如附加调试器、检测crash等

![image014](https://s2.loli.net/2022/04/10/oYTRe7hArgnLMwG.png)

远程Agent需要首先在远程目标机通过peach –a tcp启动远程代理，无需pit文件。本地peach pit文件添加如下图location，其中ip为目标机ip。

![image015](https://s2.loli.net/2022/04/10/bLp5WQyd8OEo29D.png)

可用Monitor如下图：

![image016](https://s2.loli.net/2022/04/10/9qpdJo7mRVn1hWB.png)

Windows Debugger Monitor通过windbg控制一个windows调试实例，主要参数：

- CommandLine ：运行的命令行，如下图： 

![image-20220410152250726](https://s2.loli.net/2022/04/10/W52blMGHqhCX6Lt.png)

文件fuzz时上述文件名fuzzed.wav需要与Publisher参数一致。如下图：

[![image018](https://s2.loli.net/2022/04/10/RV6uqIiEFz9aeJP.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image018.png)

- SymbolsPath：windbg符号路径。
- StartOnCall ：StateModel有匹配调用时附加调试器。
- NoCpuKill：默认false，表示当被测目标进程cpu占用为0时将其结束。

Peach3对非内核目标使用的混合调试模式，首先通过CreateProcess DEBUG_PROCESS参数创建调试进程，当检测到被测目标有感兴趣faults产生时会使用windbg的dbgeng.dll进行重现调试，最后利用windbg插件msec.dll的!exploitable命令对漏洞的可利用性进行初步判断，记录结果。

#### Test

指定使用哪个Agent、StateModel，Publisher用什么方法发送数据，使用什么方法变异数据，日志文件路径等。可以有多个Test，使用时通过peach命令行指定要运行的Test名称，未指定默认运行名称为”Default”的Test。如下图：

[![image019](https://s2.loli.net/2022/04/10/9eTRjAzH7raOyNJ.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image019.png)

Strategy（变异策略）包括：

- Random：默认会随机选择最大6个元素（可以通过参数MaxFieldsToMutate设置）利用随机mutator（变异器）进行变异。
- Sequential：Peach会顺序对每个元素使用其所有可用的Mutators进行变异。
- RandomDeterministic：Peach默认规则。这个规则对pit xml文件中元素根据Mutators

生成的Iterations链表做相对随机（由链表中元素数目决定）的顺序混淆，所以每个xml文件每次运行生成的测试用例多少、顺序固定，这样才能保证skipto的准确性。Peach3包括元素增、删、改、交换，经验值，逐位、双字等Mutators，见下图：

[![image020](https://s2.loli.net/2022/04/10/vbFlHaKwfmY65Ug.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image020.png)

### Fuzz Wav文件

- Wav文件格式

![image021](https://s2.loli.net/2022/04/10/niHuSPhmkJ1DvKM.png)

- Pit文件

[![image022](https://s2.loli.net/2022/04/10/CKqclF1kY8oBv7T.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image022.png)

[![image023](http://blog.nsfocus.net/wp-content/uploads/2015/07/image023.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image023.png)

[![image024](https://s2.loli.net/2022/04/10/w1cAtDLqVSmuRs9.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image024.png)

[![image025](https://s2.loli.net/2022/04/10/FdEkKGtINh8ygf7.png)](http://blog.nsfocus.net/wp-content/uploads/2015/07/image025.png)

