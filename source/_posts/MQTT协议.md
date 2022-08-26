---
title: MQTT协议
date: 2022-05-07 12:15:22
tags:
    - 协议
---

<!--more-->

# MQTT

​	MQTT（Message Queuing Telemetry Transport ， 消息队列遥测传输协议），是一种基于发布/订阅（publish/subscribe）模式的“轻量级”通讯协议，该协议构建与TCP/IP协议上，与 HTTP 一样，提供有序、无损、双向连接，由IBM在１９９９年发布。

![img](https://s2.loli.net/2022/05/07/GzfNEwSIOk2BP8o.png)

## MQTT协议身份

有三种身份：发布者（Publish）、代理（Broker）（服务器）、订阅者（Subscribe）。其中，消息的发布者和订阅者都是**客户端**，消息代理是**服务器**，消息发布者可以同时是订阅者。

MQTT传输的消息分为：主题（Topic）和负载（payload）两部分

    Topic，可以理解为消息的类型，订阅者订阅（Subscribe）后，就会收到该主题的消息内容（payload）
    payload，消息的内容
## MQTT消息格式

MQTT协议是应用层协议，需要借助TCP/IP协议进行传输，类似HTTP协议，MQTT协议也有自己的格式。

```
固定头部+可变头部+消息载体
```

1. 固定头部：通过固定头部区分多种消息类型，如连接，发布，订阅等

2. 可变头部：有些协议类型中存在，在有些协议中不存在

3. 消息载体：消息的主要内容

   ![image-20220507190309000](https://s2.loli.net/2022/05/07/HzVrvWo8PBlpAIO.png)

## 具体头部解析

https://baijiahao.baidu.com/s?id=1715575644678049062&wfr=spider&for=pc

### 用mosquitto测试MQTT

通过mosquitto先建立一个2233端口

![image-20220507181601515](https://s2.loli.net/2022/05/07/dchxuNizWnEtUSK.png)

接着通过 `mosquitto_sub.exe -h 127.0.0.1 -p 2233 -t "topic_test"`命令创建一个订阅者topic

![image-20220507181728788](https://s2.loli.net/2022/05/07/quMxnfc2Cj7WtXD.png)

再通过一个命令 `mosquitto_pub.exe -t "topic_test" -p 2233 -m "MESSAGE"`向2233端口发送MESSAGE

就会得到

![image-20220507181944259](https://s2.loli.net/2022/05/07/PGToBnQ3mCO6gvU.png)

### Wireshark抓包

使用Wireshark配合npcap联合抓包，搜寻端口tcp.port==2233,可以抓捕到一些流量包，因为MQTT是基于http的，所以可以看到前三个是TCP的三次握手连接

![image-20220507183645301](https://s2.loli.net/2022/05/07/zZnXTNGPIhKSH7w.png)

