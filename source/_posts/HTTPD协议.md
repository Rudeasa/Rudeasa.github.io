---
title: HTTPD协议
date: 2022-05-05 10:13:39
tags:		
    - 协议
---

<!--more-->

# httpd协议

HTTP（HyperText Transfer Protocol）超文本传输协议

## 客户端和服务端通信

![image-20220505101629316](https://s2.loli.net/2022/05/05/UEud1wQSTgyb6l2.png)

### TCP三次握手

复习一下TCP三次握手

![image-20220505102905247](https://s2.loli.net/2022/05/05/hAc9U15eYwkQvCX.png)

第一次握手：建立连接时，客户端发送syn包(syn=X)到服务器，并进入SYN_SEND状态，等待服务器确认。

第二次握手：服务器收到syn包，必须确认客户的SYN（ack=X+1），同时自己也发送一个SYN包（syn=Y），即SYN+ACK包，此时服务器进入SYN_RECV状态。

第三次握手：客户端收到服务器的SYN＋ACK包，向服务器发送确认包ACK(ack=Y+1)，此包发送完毕，客户端和服务器进入ESTABLISHED状态，完成三次握手。

完成三次握手，客户端与服务器开始传送数据


## httpd工作流程

![image-20220505103403009](https://s2.loli.net/2022/05/05/CjmBVlYPsME2TL4.png)

## http请求、响应报文格式

|       **请求头**       |         响应行         |
| :--------------------: | :--------------------: |
|       **请求行**       |       **响应头**       |
| 空行（表示请求体开始） | 空行（表示响应体开始） |
|       **请求体**       |       **响应体**       |
| 空行（表示请求体结束） | 空行（表示响应体结束） |

### http请求报文

![image-20220505104244101](https://s2.loli.net/2022/05/05/SgcTaGd65E4rfZ7.png)

### http响应报文

![image-20220505104634852](https://s2.loli.net/2022/05/05/piOH1cZArStjDq5.png)

## http 常见请求方法

```
1、GET：请求指定页面信息，并返回实体主体；
 
2、POST：向指定资源提交数据并进行处理请求，数据被包含在请求体中，POST请求可能会导致新的资源的建立或已有资源的修改；
 
3、HEAD：类似GET请求，只不过返回的响应中没有具体内容，用于获取报头；
 
4、PUT：从客服端向服务器传送的数据取代指定的文档内容；
 
5、DELETE：请求服务器删除指定的内容；
 
6、CONNECT：HTTP1.1协议中预留给能够将连接改为管道方式的代理服务器；
 
7、TRANCE：回显服务器收到的请求，主要用于测试或诊断；
```

## http 常见header

```
Host: www.test.com/ //请求的目标域名和端口号
Origin: http://localhost:8081/ //请求的来源域名和端口号 （跨域请求时，浏览器会自动带上这个头信息）
Referer: https:/localhost:8081/link?query=xxxxx //请求资源的完整URI
User-Agent //浏览器信息
Cookie: //当前域名下的Cookie
Accept: text/html,image/apng //代表客户端希望接受的数据类型是html或者是png图片类型
Accept-Encoding: gzip, deflate //代表客户端能支持gzip和deflate格式的压缩
Accept-Language: zh-CN,zh;q=0.9 //代表客户端可以支持语言zh-CN或者zh
Connection: keep-alive //告诉服务器，客户端需要的tcp连接是一个长连接
If-None-Match //如果内容未改变返回304代码，对应Etag
If-Modified-Since //对应last-midified，未被修改则返回304代码
Date: //服务端发送资源时的服务器时间
Expires: //缓存过期时间
Cache-Control: no-cache // 缓存方式
Etag // 文件内容hash
Last-Modified //最近一次文件修改时间
Content-Type: text/html; charset=utf-8 //编码格式
Content-Encoding: gzip //采用gzip对资源进行解码
Connection: keep-alive //tcp是长连接
Set-Cookie //设置Http Cookie
```

