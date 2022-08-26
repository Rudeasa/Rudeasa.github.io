---
title: 0day安全软件漏洞学习第五弹
date: 2022-04-10 16:12:26
tags:
    - 漏洞挖掘
---

<!--more-->

​		上回算对peach fuzz的Peach Pit文件有了基本的了解，可能搜集的资料并不涵盖所有点，那就今后在使用的过程中再查漏补缺，这回试试跟着教材一起通过使用Peach 对PNG文件进行Fuzz测试。

# 用Peach Fuzz PNG文件 	

首先来看一下PNG的文件格式

![image-20220410163641973](https://s2.loli.net/2022/04/10/V3CA8fQU2cKGs7a.png)

一个PNG文件最前面是8个字节的PNG签名，十六进制值为89 50 4E 47 0D 0A 1A 0A。随后是若干个数据区块(Chunk)，包括IDHR、IDAT、IEND等。

![image-20220410163725105](https://s2.loli.net/2022/04/10/QuafForTJ94KVEY.png)

由此，可将Chunk的DataModel定义如下：

```
<DataModel name="Chunk"> 
<Number name="Length" size="32" signed="false">  
	<Relation type="size" of="Data" /> 
</Number>
<Block name="TypeAndData">  
 	<String name="Type" size="4"/>  
 	<Blob name="Data" /> 
</Block>  
<Number name="CRC" size="32">  
 	<Fixup class="checksums.Crc32Fixup">     
 		<Param name="ref" value="TypeAndData"/>  
 	</Fixup> 
 </Number> 
 </DataModel> 
```

首先我们定义三个数据模型：Length、Type、CRC。

首先，不去考虑每个Chunk的具体结构，而将PNG简单地认为是由一个PNG签名和若干结构相同的Chunk组成。那么可以在Chunk数据模型之后将PNG文件的DataModel进行如下

```
 <DataModel name="Png"> 
 	<Blob name="pngMagic" isStatic="true" valueType="hex" value="89 50 4E 47 0D 0A 1A 0A" /> 
 	<Block ref="Chunk" minOccurs="1" maxOccurs="1024"/> 
 </DataModel> 
```

minOccrus=”1” maxOccurs=”1024” 表示该区块最少重复1次，最多重复1024次。 然后开始配置StateModel：第一步需要修改文件生成畸形文件；第二步需要把该文件关闭；第三部需要调用适当的程序打开生成的畸形文件。在这里，我们使用pngcheck（http://www.libpng.org/pub/png/apps/pngcheck.html）来作为打开畸形文件的程序。StateModel定义如下：	

```
<StateModel name="TheState" initialState="Initial">         
 	<State name="Initial">                                 
 		<!-- Write out our png file -->                 
 		<Action type="output"> 
        <DataModel ref="Png"/>                         
        <!-- This is our sample file to read in -->                         
        <Data name="data" fileName="sample.png"/>                 
    </Action>                 
    <!-- Close file -->                 
    <Action type="close"/>                                 
    <!-- Launch the target process -->                 
    <Action type="call" method="D:\pngcheck.exe">                         
    	<Param name="png file" type="in">                                 
    		<DataModel ref="Param"/>                                 
    		<Data name="filename">                                       
    			<!-- Name of Fuzzed output file -->                                       
    			<Field name="Value" value="peach.png"/>                                 
    		</Data>                         
    	</Param>                 
    </Action>         
</State> 
</StateModel> 
```

在call动作中我们引用了一个叫做“Param”的数据模型，这个数据模型用来存放传递给pngcheck.exe的参数，即畸形文件的文件名。所以“Param”需要包含一个名为“Value”的字符型静态数据。我们需要在StateModel之前定义该数据模型。 

```
<DataModel name="Param">         
	<String name="Value" isStatic="true" /> 
</DataModel> 
```

然后需要在Test元素中配置Publisher信息。在这里需要使用FileWriterLauncher，它能够在写完文件之后使用call动作启动一个线程。它的参数应当是生成的畸形文件。 

```
<Test name="TheTest">    
	<StateModel ref="TheState"/>    
	<Publisher class="file.FileWriterLauncher">  
		<Param name="fileName" value="peach.png"/> 
	</Publisher> 
</Test> 
```

最后在Run信息配置中指定要运行的测试名称。 

```
<Run name="DefaultRun">  
	<Test ref="TheTest" />  
</Run>  
```

至此，Peach Pit文件就配置完毕了。将其命名为png_dumb.xml并和sample.png保存在Peach目录下。运行Fuzzer： 

![image-20220411144000918](https://s2.loli.net/2022/04/11/SUXk9WD8Gh4Zawe.png)

我的fuzz可能版本与书中的不兼容，调试了好久还是报错很多，可能还是需要今后有能力再来回头解决这问题，接下来就跟着书上走了。

### 完整xml代码：

```
<?xml version="1.0" encoding="utf-8"?>
<Peach xmlns="http://peachfuzzer.com/2012/Peach" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://peachfuzzer.com/2012/Peach ../peach.xsd">
<DataModel name="Chunk"> 
	<Number name="Length" size="32" signed="false">  
		<Relation type="size" of="Data" /> 
	</Number> 
	<Block name="TypeAndData">  
		<String name="Type" size="4"/>  
		<Blob name="Data" />

	</Block>  
	<Number name="CRC" size="32">  
		<Fixup class="checksums.Crc32Fixup">     
			<Param name="ref" value="TypeAndData"/>  
		</Fixup> 
	</Number> 
</DataModel> 
<DataModel name="Png"> 
	<Blob name="pngMagic" isStatic="true" valueType="hex" value="89 50 4E 47 0D 0A  1A 0A" /> 
 	<Block ref="Chunk" minOccurs="1" maxOccurs="1024"/> 
</DataModel> 

<StateModel name="TheState" initialState="Initial">         
	<State name="Initial">                                 
		<!-- Write out our png file -->                 
		<Action type="output"> 
   <DataModel ref="Png"/>                         
    <!-- This is our sample file to read in -->                         
    <Data name="data" fileName="sample.png"/>                 
</Action>                 
<!-- Close file -->                 
<Action type="close"/>                                 
<!-- Launch the target process -->                 
<Action type="call" method="D:\peach fuzz\pngcheck.exe">                         
		<Param name="png file" type="in">                                 
			<DataModel ref="Param"/>                                 
			<Data name="filename">                                       
			<!-- Name of Fuzzed output file -->                                       
				<Field name="Value" value="peach.png"/>                                 
			</Data>                         
		</Param>                 
	</Action>         
</State> 
</StateModel> 
<DataModel name="Param">         
	<String name="Value" isStatic="true" /> 
</DataModel> 
<Test name="TheTest">    
<StateModel ref="TheState"/>    
	<Publisher class="file.FileWriterLauncher">  
	<Param name="fileName" value="peach.png"/> 
</Publisher> 
</Test> 
<Run name="DefaultRun">  
	<Test ref="TheTest" />  
</Run>  
</Peach>
```

正常的运行结果：程序会根据Peach Pit文件的配置以及输入的样本sample.png生成畸形文件，并将该畸形文件传到pngcheck.exe中。

![image-20220411144422044](https://s2.loli.net/2022/04/11/pf9wH2qQvbsOcxZ.png)

### 变异测试

接下来对png_dumb.xml做一些改动，让程序调用Windows资源管理器打开畸形文件。 首先，在StateModel的Action中找到type=”call”的这一行，并将后面的pngcheck地址改为explorer。 

```
<Action type="call" method="explorer"> 
```

然后在Publisher配置中将class改为file.FileWriterLauncherGui，并且为Publisher增加一个名为WindowName、值为peach.png的参数。 

```
<Publisher class="file.FileWriterLauncherGui"> 
<Param name="fileName" value="peach.png"/> 
<Param name="WindowName" value="peach.png"/>
</Publisher> 
```

FileWriterLauncherGui和FileWriterLauncher的区别在于，前者用于运行带界面的GUI程序，并且在运行后会自动关闭窗口标题中含有WindowName的值的GUI窗口。 执行Fuzzer，可以看到生成的各个PNG畸形文件逐一地被explorer打开，如图示。 为了捕获程序的异常，还需要配置一下Agent and Monitor，调用WinDbg进行调试。在这之前，请确认已经安装了WinDbg 6.8。 

![image-20220411145437995](https://s2.loli.net/2022/04/11/UTIye1KGE5h2CkA.png)

首先，将StateModel中最后一个Action删掉，并添加这一行： 

```
<Action type="call" method="ScoobySnacks" /> 
```

然后，在StateModel下面加入Agent配置： 

```
<Agent name="LocalAgent"> 
<Monitor class="debugger.WindowsDebugEngine">  
	<Param name="CommandLine" value="explorer peach.png" />  	 <Param name="StartOnCall" value="ScoobySnacks" /> </Monitor>  
<Monitor class="process.PageHeap">  
	<Param name="Executable" value="explorer"/> 
</Monitor> 
</Agent> 
```

然后，在Test配置的第一行加入： 

```
<Agent ref="LocalAgent"/> 
```

并且在Publisher的最后一行加入名为debugger，值为true的参数： 

```
<Param name="debugger" value="true" /> 
```

最后在Run配置的Test元素后面加入日志配置： 

```
<Logger class="logger.Filesystem"> 
<Param name="path" value="logs"/> 
</Logger> 
```

重新运行Fuzzer，如图17.2.5所示，Fuzzer程序启动了一个Local Peach Agent，通过该Agent控制WinDbg进行调试并捕获异常事件。 

![image-20220411145352952](https://s2.loli.net/2022/04/11/qUEhGudCO1oWv3n.png)

虽然自己还没能成功实践过，只能看着书记录一下过程，等今后解决自己peach版本兼容问题，再来重新走一遍。

