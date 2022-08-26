---
title: httpd实例操作
date: 2022-05-05 11:13:14
tags:
    - httpd
---

<!--more-->

记录一下通过ncnc命令来简易地学一下http协议的一部分实战技巧

### nc常用命令

```
1. -4 强制使用ipv4
2. -6 强制使用ipv6
3. -D 允许socket通信返回debug信息
4. -d 不允许从标准输入中读取
5. -h 显示nc帮助文档
6. -i interval 指定每行之间内容延时发送和接受，也可以使多个端口之间的连接延时
7. -c  shell 命令为“-e”；使用/bin/sh执行[危险！！]
8. -k 当一个连接结束时，强制nc监听另一个连接。必须和-l一起使用
9. -l 用于监听传入的数据链接，不能与-p -z -s一起使用。-w 参数的超时也会被忽略
10. -n 不执行任何地址，主机名，端口或DNS查询
11. -v 输出详细报告[使用两次更详细]
12. -z 只监听不发送任何包
13. -p  port  端口 本地端口号
```

# 本地测试连接

## 终端与终端

先随便创建一个本地端口8899

![image-20220505144733184](https://s2.loli.net/2022/05/05/mXCv7xiY4f8kV2J.png)

然后另起一个终端监听这个8899端口，因为是本地端口，可以直接通过命令`nc 127.0.0.1 8899`监听

![image-20220505144925258](https://s2.loli.net/2022/05/05/6On1YcS2xyfEdM5.png)

### 连接与监听成功

![image-20220505145206907](https://s2.loli.net/2022/05/05/StAjVYiX521PwnK.png)

这时候可以看到不管我们在本地端口输入什么，监听端口也都能返回一样的字符串。

## 终端与浏览器

这时候我们不再使用另一个终端nc连接127.0.0.1 8899端口，而是打开浏览器访问127.0.0.1：8899，可以看到直接在终端返回了一些http的header详细信息，得到是request包。

![image-20220505111511034](https://s2.loli.net/2022/05/05/mRTgq5wOxnQsUEl.png)

**request：**

```
GET / HTTP/1.1
Host: 127.0.0.1:8899
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:73.0) Gecko/20100101 Firefox/73.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```

但是http是一来一回的，还有response包，当我们把以下response包在终端发送

**response：**

```
HTTP/1.1 200 OK 
Content-Type:text/html;charset=UTF-8
Content-Length:927
Connection:close
Cache-control:no-cache

<html>    
<body>
<h1>hello!!!!</h1>
</body>
</html>
```

### 连接与监听成功

这些response包会被发送到浏览器解析，通过解析我们可以在浏览器看到结果

![image-20220505112431921](https://s2.loli.net/2022/05/05/KYGhyC6TX3e54uD.png)

这些都是静态网页的本地测试，需要人工弄好每一步

## 通过程序反弹

### 直接shell反弹

`nc -lvvp 9999 -e /bin/bash`这个命令就是通过本地的/bin/bash这个程序会返回到本地的9999端口，不太好理解。

![image-20220505144537414](https://s2.loli.net/2022/05/05/olAyFrp1VUjzT94.png)

反正就是差不多当用户另起一个终端监听9999端口，如上图右侧，就可以输入一些shell命令，因为右侧的命令是通过左侧的端口以/bin/bash形式执行，所以右侧输入`ls`即可查看本地所有文件信息。

这样的命令平时可以运用在web的一些远程渗透。

### 本地网页反弹

先创建两个文件：cgi.sh和index.html

**cgi.sh:**

```
#! /bin/bash
echo "HTTP/1.1 200 OK					
"
cat index.html
echo"

"
```

cgi.sh的内容根据上篇文章的http响应报文格式编写，其实**主要就是调用本地index.html文件里的信息，这就是cgi（通用网关处理）的作用**，专门用来处理这些请求的。

**index.html：**

```
<html>    
<body>
<h1>hello!!!!</h1>
</body>
</html>
```

index.html就是上面response包的主体信息（网页源码），如果将其替换就会显示不同内容

![image-20220505152503954](https://s2.loli.net/2022/05/05/4Hq2egMd9PrDt1l.png)

将两个文件放在同一目录下，使用命令 `nc -lvvp 9999 -e cgi.sh`另一终端监听，返回如下

![image-20220505153255028](https://s2.loli.net/2022/05/05/k2UIr34PwzyQLZ5.png)

或者不用终端监听，用浏览器打开127.0.0.1:9999页面

![image-20220505153453477](https://s2.loli.net/2022/05/05/V3LTprNnDZ2dhi4.png)

都可以成功回显

