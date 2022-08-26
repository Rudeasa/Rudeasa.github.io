---
title: soap协议
date: 2022-05-05 16:05:48
tags:
    - 协议
---

<!--more-->

# SOAP

SOAP（Simple Object Accrss Protocol，简单对象访问协议）是一种简单的**基于XML**的协议，可以使应用程序在分散或分布式的环境中通过HTTP，STMP等协议来交互信息

### XML

- XML：一种可扩展标记语言，被设计用来传输和存储数据。（和html很相似）
- XML命名空间 ：用来避免元素命名冲突

简单示例

```
<?xml version="1.0" encoding="UTF-8"?>
<info>	
<name>Nick</name>
<age input="num">11</age>
<sex>male</sex>
</info>
```

## 语法规则

1. SOAP消息必须用XML来编码

2. SOAP消息必须使用SOAP Envelope命名空间

3. SOAP消息必须使用SOAP  EncodingStyle命名空间

4. SOAP消息不能包含DTD引用

5. SOAP消息不能包含XML处理指令

   ## SOAP消息基本结构

![image-20220505170752993](https://s2.loli.net/2022/05/05/xw8Fq673Aeg5p21.png)

## SOAP报文实例

这是用wireshark抓取的一个SOAP报文，可以看到还是基于HTTP协议的

![image-20220505173036240](https://s2.loli.net/2022/05/05/KfHYwpdsvtB1mWU.png)

SOAP报文中的<body>由于是开发者自己定义的,我们也是随机抓取的,对其内容就不太了解了.

