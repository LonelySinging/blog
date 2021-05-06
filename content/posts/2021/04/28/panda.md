---
title: "熊猫烧香病毒分析"
date: 2020-03-15T22:14:22+08:00
tags : ["病毒","逆向分析","IDA","PEID","火绒剑"]
categories : ["逆向分析"]
---

# 概述

弄到了一个熊猫烧香的样本，Delphi程序，和前面的彩虹猫根本就不是一码事儿，这次只能完全看汇编代码了，不过伪代码可以帮忙分析循环条件什么的。



> 环境工具:
> win7 x64 , 火绒剑 IDA OD IDR

# PEID

---

![image-20200315141531736](https://qiniusave.5cser.com/img/image-20200315141531736.png)

Delphi的程序 但是以前没有写过也没有逆向过Delphi程序

# 直接运行

---

先开着火绒剑，直接运行一下

先设置好虚拟机环境，把`文件后缀名`,`显示隐藏文件`也打开，这样就能显示所有东西了

![image-20200315143342865](https://qiniusave.5cser.com/img/image-20200315143342865.png)

1. 任务管理器被关闭了，再打开的时候已经打不开了
2. 另外，火绒剑也没了... 卧槽，这个不是老病毒了么，怎么火绒剑还会被关闭？
3. 进入C盘，因为设置了隐藏文件可见的原因，看到了这两个文件，从大小来看，初步判断setup.exe就是病毒本身副本，autorun.inf的目的就是为了让`开启了自动播放`的用户打开磁盘的时候运行setup.exe
   1. ![image-20200315141929633](https://qiniusave.5cser.com/img/image-20200315141929633.png)
   2. ![image-20200315142139944](https://qiniusave.5cser.com/img/image-20200315142139944.png)

4. 稍后桌面出现了一个ini文件
   1. ![image-20200315143725176](https://qiniusave.5cser.com/img/image-20200315143725176.png)
   2. 内容是感染日期，不过听说病毒会设置隐藏文件不可见... 但是我怎么能看到呢
5. 于是去看了文件夹设置，发现单选框没有被选择上，设置显示所有隐藏文件
6. 结果桌面一个刷新所有的隐藏文件都看不到了，再进入设置，发现被改回来了
   1. ![image-20200315144108599](https://qiniusave.5cser.com/img/image-20200315144108599.png)
   2. 再也改不回来了
7. 看到exe文件图标都变成 熊猫烧香的图标了
8. 虽然不能看到隐藏文件，但是还有办法
   1. ![image-20200315144314101](https://qiniusave.5cser.com/img/image-20200315144314101.png)
   2. 通过 `dir /a` 命令 还是能看到被隐藏起来的 `Desktop_.ini` 文件
9. 因为火绒剑被关闭了的原因，已经不能知道更多信息了，但是为什么一个老病毒能关闭火绒剑呢？一般不都是黑名单方式检测，然后关闭，这病毒出来的时候火绒剑还没有出现的吧。。。
10. 不过后面想想，这个病毒似乎已经开源了，很多人都写了，所以我手上的样本不一定是原版，有人给他升级了？

# 字符串查看

---

因为是Delphi程序的关系，这里需要IDR来查看字符串

![image-20200315145550526](https://qiniusave.5cser.com/img/image-20200315145550526.png)

1. 这里可以看到很多奇奇怪怪的字符串
2. 有关于文件名的，还有一个cmd命令，是应该打开分享的
3. 注册表路径
4. 还有一些弱口令
   1. ![image-20200315145744696](https://qiniusave.5cser.com/img/image-20200315145744696.png)
5. 以及各种进程名
   1. ![image-20200315145837101](https://qiniusave.5cser.com/img/image-20200315145837101.png)
   2. 但是我没有找到火绒啊(/(ㄒoㄒ)/~~) 所以为什么会被干掉
6. 还有文件路径，以及一大堆的乱七八糟的字符，看着像乱码
7. 那么就可以猜测，
   1. 他会关闭一些安全软件
   2. 修改注册表，实现隐藏自己的目的
   3. 通过弱口令对什么东西测试，实际上是局域网分享的密码吧

# IDA分析

---

> 才注意到 win7 32位不能运行IDA7.0 所以使用了6.8版本

## 导入函数

> 导出函数太多了，就找了几个比较有代表性的

```
00410184  CreateThread kernel32
```

+ 这里肯定可以创建线程，多线程病毒

```
00410188  WriteFile kernel32
00410190  SetFilePointer kernel32
00410194  SetEndOfFile kernel32
0041019C  ReadFile kernel32
004101A8  GetFileSize kernel32
004101AC  GetFileType kernel32
```

+ 操作文件（这些应该是用来实现文件感染的）

```
004101FC  RegSetValueExA advapi32
00410200  RegOpenKeyExA advapi32
00410204  RegDeleteValueA advapi32
00410208  RegCreateKeyExA advapi32
00410298  CopyFileA kernel32
```

+ 注册表操作

```
00410224  WinExec kernel32
```

+ 执行命令或者创建文件

```
00410240  GetWindowsDirectoryA kernel32
0041024C  GetSystemDirectoryA kernel32
```

+ 对系统目录会有什么操作

```
00410218  AdjustTokenPrivileges advapi32
00410214  LookupPrivilegeValueA advapi32
```

+ 很多病毒都会有的提权操作

```
00410274  FindNextFileA kernel32
00410278  FindFirstFileA kernel32
```

+ 文件遍历

```
004102AC  WNetAddConnection2A mpr
004102A8  WNetCancelConnectionA mpr
004102E8  WSACleanup wsock32
00410308  connect wsock32
004102F8  socket wsock32
······

00410314  InternetGetConnectedState wininet
00410318  InternetReadFile wininet
0041031C  InternetOpenUrlA wininet
······
```

+ 网络操作

```
00410334  DeleteService advapi32
0041032C  OpenServiceA advapi32
00410330  OpenSCManagerA advapi32
00410338  ControlService advapi32
······
```

+ 操作服务

```
00410350  URLDownloadToFileA URLMON
```

+ 应该会下载什么东西，执行？

```
00410228  TerminateProcess kernel32
```

+ 结束进程

```
004102D8  FindWindowA user32
```

+ 应该是用来遍历窗口了，但是没有看到其他的函数



由此可以看出，这个病毒功能复杂，也基本可以知道一些大概行为

链接网络之后应该还会有其他操作，不过服务器应该已经关闭了

## 导入额外信息

之前反编译彩虹猫的时候，按下F5 就能看到伪代码，而且基本上甚至都能运行。

但是这次的Delphi程序，按下F5之后，却是一坨翔：大量的未知函数，还有很多参数明显错误的函数参数。这可让我头疼，所以百度了一下解决方法，

+ 设置符号,告诉IDA这个就是Delphi程序，告诉具体版本（这里我发现设置所有delphi程序版本能识别更多的delphi函数）
+ IDR能更好的识别代码中的函数名称，所以，下载IDR，导出IDC，然后又能识别出一些函数名称

执行IDC之后，整个函数风格都变了，还出现了很多的字符串常量，这就贼棒了

但是 反编译的伪代码依旧是一坨翔，有些能看出是很明显的错误

这就没办法了，只能死磕汇编代码了，所幸我还会些汇编

## IDA分析开始

---

### 主函数

![image-20200315153459262](https://qiniusave.5cser.com/img/image-20200315153459262.png)

1. 首先验证了字符串，其实没看懂这步是干嘛的，还对字符串进行了加密（难道是为了避免特征值?）
2. 接着就是三个函数
3. 第一个函数 sub_4082F8
   1. 把自己复制到 `C:\Windows\System32\drivers\spcolsv.exe` (似乎还做了一定的校验)
4. 第二个函数 sub_40CFB4
   1. ![image-20200315154633174](https://qiniusave.5cser.com/img/image-20200315154633174.png)
   2. sub_40A7EC 创建一个线程用来感染文件
   3. sub_40C5B0 是一个定时器，每六秒钟执行一次，功能是保证setup.exe autorun.inf文件的存在
   4. sub_40BD08 会扫描局域网，如果可能，就把远程主机磁盘映射到本地
5. 第三个函数 sub_40CED4 里面有四个定时器，但是最后一个定时器函数没解析出来
   1. 定时器一 1s
      1. 创建新的线程，遍历窗以及线程，在黑名单中的全部关闭杀死
      2. 注册注册表键值，实现隐藏文件，以及自启动
   2. 定时器二 1200s
      1. 打开`https://wangma.9966.org/down.txt` 获取文本文件，并执行其中的命令
   3. 定时器三 创建了两个线程 10s
      1. 线程一 把定时器二的函数又执行了一遍
      2. 线程二 执行了 `cmd.exe /c net share admin$ /del /y` 命令 关闭了分享（为啥会关闭分享呢 不应该打开让人感染吗((・∀・(・∀・(・∀・*))）

### 第一个主要函数 sub_4082F8

1. 判断当前目录下是否有`Desktop_.ini`文件，如果有就删除
2. 获取自身文件名，判断是否是副本(也就是位于`C:\Windows\System32\drivers\spcolsv.exe`)
3. 如果不是副本，那么就复制自己到 `C:\Windows\System32\drivers\spcolsv.exe`
4. 或者启动方式是被感染文件，则会通过感染时追加到文件末尾的原始文件大小，推算出病毒本体的大小随即从被感染文件中剥离出病毒本体，写入到`C:\Windows\System32\drivers\spcolsv.exe`，然后执行，之后本线程结束
5. 在剥离病毒本体的过程中，其实有把文件给还原了，只不过释放出了一个bat文件又删除掉了



### 第二个主要函数 sub_40CFB4

#### 文件感染 sub_40A7EC

1. 获取可用磁盘 类型为
   1. 硬盘
   2. 移动磁盘
   3. 网络驱动器
2. 跳过 分区为A和B的分区 （软盘分区）
3. 开始遍历文件感染
   1. GHO文件 直接删除
   2. EXE、SCR(屏幕保护程序)、COM、PIF文件会感染，感染方式见下文
   3. htm、html、asp、php、jsp、aspx文件则会末尾追加内容，具体内容分析见下文
   4. 对于白名单的文件夹跳过，具体白名单见下文
4. 其中，感染过程会被记录在 `C:\test.txt`文件中，但是我并没有看到这个文件

#### 自动运行保活 sub_40C5B0

每隔六秒执行一次下面的逻辑

![image-20200315174758220](https://qiniusave.5cser.com/img/image-20200315174758220.png)

#### 搜索局域网主机 sub_40BD08

遍历整个局域网，如果主机可用，尝试映射网络驱动器到本地，如果没有网络就一直等待



### 第三个主要函数 sub_40CED4

#### 结束安全软件 sub_40CD30

1. 通过函数 sub_406F3C 创建了一个线程，功能如下
   1. 遍历所有窗口，获取他们的标题，如果含有以下内容，则发送 `WM_QUIT`消息试图关闭它
      1. ![image-20200315175949777](https://qiniusave.5cser.com/img/image-20200315175949777.png)
   2. 还会通过虚拟按键的方式关闭安全软件
   3. 终结以下下进程
      1. ![image-20200315180231742](https://qiniusave.5cser.com/img/image-20200315180231742.png)
   4. 编辑注册表键值 实现自启动，以及文件隐藏
      1. `Software\Microsoft\Windows\CurrentVersion\Run`
      2. `SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced\Folde`

#### 获取网络命令执行 sub_40CE8C

> 打开 `http://wangma.9966.org/down.txt`链接，获取一个文本，并执行里面每一行命令
>
> 但是因为这个网址已经不能访问了，所以具体不知是什么
>
> 根据Wiki百科的内容来看，大概是用来给网站刷流量的
>
> 反正都能执行命令了，还有啥不能干的



#### 关闭共享 sub_40CE94

> 创建了两个线程，但是第一个线程只是把上一个定时器的函数执行了一遍而已

+ 关闭共享
  + 执行命令: `cmd.exe /c net share admin$ /del /y`

#### 未知的定时器函数

![image-20200315181029046](https://qiniusave.5cser.com/img/image-20200315181029046.png)

这里可能还是一个定时器函数，但是IDA没有解释出来，具体是什么功能暂时不知道



# 附录

---

## 可执行文件感染

---

经过对比，发现对于一个可执行文件而言，病毒只是把自己插入到可执行文件的前面（覆盖了一部分内容），在文件末尾追加了一个字符串 由此来标记这个文件已经被感染了，

最后跟了一个被感染文件的大小

![image-20200315182935175](https://qiniusave.5cser.com/img/image-20200315182935175.png)



![image-20200312164829044](https://qiniusave.5cser.com/img/image-20200312164829044.png)

我以为会添加节区啊啥的... 居然如此暴力, 学到了 啧~



## 前端脚本感染

会在 htm、html、asp、php、jsp、aspx 文件后面追加内容，我在C盘根目录下放一个空的html文件骗到内容如下

![image-20200315195516581](https://qiniusave.5cser.com/img/image-20200315195516581.png)

URL现在指向一个。。。 广告页面？

![image-20200315195821012](https://qiniusave.5cser.com/img/image-20200315195821012.png)

![image-20200315200020927](https://qiniusave.5cser.com/img/image-20200315200020927.png)

> panda VPN? 怕不是病毒作者的新网站？



# 其他的

## Whboy 是什么

“Whboy” 这个字符串多次出现，我一直好奇是什么意思，去了趟wiki百科，找到了含义，就是“武汉男生”




有人通过对此毒脱壳后的特征码分析发现有“whboy”的标识[6]，而此标识也曾出现在2004年的一只病毒“武汉男生”上，所以该病毒也被称为“武汉男生”，通过查看李俊的早期作品可以看到他的QQ号码以及他创建的网站信息，有了这些信息，侦破案件的湖北公共信息网络安全监察的工作就容易了许多。

该病毒作者是李俊（1982年-）[7]，武汉新洲区人，据他的家人以及朋友介绍，他在初中时英语和数学成绩都很不错，但还是没能考上高中，中专在娲石职业技术学校就读，学习的是水泥工艺专业，毕业后曾上过网络技术职业培训班，他朋友讲他是“自学成才，他的大部分电脑技术都是看书自学的”[8]。2004年李俊到北京、广州的网络安全公司求职，但都因学历低的原因遭拒，于是他开始抱着报复社会以及赚钱的目的编写病毒了。他曾在2003年编写了病毒“武汉男生”，2005年他还编写了病毒QQ尾巴，并对“武汉男生”版本更新成为“武汉男生2005”。




## win7 x64不发作

> 我最开始的虚拟机是 win7 x64 运行之后，主程序直接退出，生成的`spcolsv.exe`文件也不执行，所以最终我安装了 32位的win7虚拟机

## OD导入Map闪退

> 知道IDR可以导出Map文件 帮助OD识别函数，但是我下载的OD导入Map文件就闪退，换了鱼C的 吾爱的都是一样闪退，删除了所有插件也闪退。。。为啥啊...
>
> 不过所幸多看了一眼IDR的菜单栏发现了IDC，结果比IDA识别出来的函数名更人性化一点



## delphi 函数调用约定

![image-20200308003405472](https://qiniusave.5cser.com/img/image-20200308003405472.png)

