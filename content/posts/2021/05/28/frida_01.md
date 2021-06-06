---
title: "使用Frida完成android的hook"
date: 2021-05-28T16:13:23+08:00
draft: false
tags: ["反编译","HOOK","Frida","Android"]
categories: ["逆向分析"]
---



# Frida

---

先喊一句 **Frida牛逼**再开始后面的内容

一次偶然在别人的博客注意到了Frida这个工具，是一个android的hook工具。发现用它可以实现一些很厉害的功能，比如破解一些软件功能什么的。但是，我只是想实现一个简单的功能罢了。

我一直在用一个图片软件，可以浏览很多图片，但是软件没有下载功能。所以想要弄到图的话，会很麻烦。抓包分析api这样的手段当然可以使用，但是我想做的事情是想给软件增加一个功能，实现批量下载。

这篇文章是用来记录我第一次遇到Frida的使用过程，以免将来再次使用的时候忘记了怎么使用，比如今天准备反编译apk的时候忘记了apktool怎么使用了。

这次使用Shaft作为分析对象。这个软件是一个能在国内看P站图的软件，非常nice。并且是开源的。

[MrTrying/Pixiv-Shaft: Pixiv第三方Android客户端 (github.com)](https://github.com/MrTrying/Pixiv-Shaft)



# 反编译APK

---

首先通过反编译的方式，先了解一下软件的结构，以及是否加固了。

## 得到dex文件

众所周知，APK的本质就是zip文件，所以我们通过更改apk文件后缀名为.zip，使用压缩软件解压的方式得到dex文件。

![image-20210528162754720](https://qiniusave.5cser.com/mdimage-20210528162754720.png)

可以看到解压之后有classes.dex文件，dex文件就是apk文件真正的可执行文件，具体的代码逻辑都在这个里面，其他的不是此次的重点。另外提一嘴，如果想要得到apk的AndroidManifest.xml文件，需要使用apktool工具，通过查看这个文件可以得到apk需要的权限，以及入口类在哪里。不过直接解压得到的AndroidManifest.xml文件是加密的，这时候就可以使用apktool工具解包apk，能够得到AndroidManifest.xml明文，并且apktool直接把dex反编译成了smali文件(相当于java的汇编)。但是实际上，我认为这是一个令人智熄的操作。因为已经有了更好的解决方法

[Apktool - A tool for reverse engineering 3rd party, closed, binary Android apps. (ibotpeaches.github.io)](https://ibotpeaches.github.io/Apktool/)

## dex2jar

接下来通过dex2jar工具把dex转换为jar包。工具的下载链接如下

[pxb1988/dex2jar: Tools to work with android .dex and java .class files (github.com)](https://github.com/pxb1988/dex2jar)

![image-20210528165200296](https://qiniusave.5cser.com/mdimage-20210528165200296.png)

只需要把dex文件拖到dex2jar.bat文件上，就能得到类似于`classes-dex2jar.jar`这样的jar包了

不过这个过程不会很顺利，有时候也会生成一个错误文件，这时候就是反编译出错了。可以尝试更新最新的版本解决这个问题，但是我这次更新完了之后也还是不行。但是好在需要的代码都成功反编译出来了



## 查看java代码

此时就需要用到另一个工具了就是`jd-gui` 全称应该是java反编译器

通过它可以直接看到jar包中的java代码。很神奇的是，即使是通过kotlin编写的程序也能通过它看到java代码，所以我还是比较疑惑，kotlin和java之间的关系。或者，其实他们之间的关系就像是delphi被IDA反编译成C语言代码一样的关系？

工具链接[Java Decompiler (java-decompiler.github.io)](https://java-decompiler.github.io/)

![image-20210528173409317](https://qiniusave.5cser.com/mdimage-20210528173409317.png)

直接将代码拖到jd-gui的窗口上，就能看到代码了。经过jd-gui反编译的代码非常清晰，甚至保留了符号名这样非常有利于分析程序代码。

但是大多时候并不会很顺利，还会有加固的可能。(在windows上叫做加壳)

一般来说，你看到dex文件很小的时候就大概率是加固的程序。这时候就需要通过特殊手段去壳之后才能得到真正的dex。例如通过xposed框架去除加固。

此次特意选择了一个没有加固的程序用来学习Frida。

## 找到有价值的函数

我决定这次对下载函数进行拦截，获取到下载链接

经过浏览代码。我看到了以下几个类

![image-20210528174400384](https://qiniusave.5cser.com/mdimage-20210528174400384.png)

大概需要的函数就在这里了，上面一些类似乎并没有正常的反编译下来，不清楚是通过什么手段实现的。简单浏览代码之后，目光锁定到了下面这个函数

```java
  public static String getShowUrl(IllustsBean paramIllustsBean, int paramInt) {
    return (paramIllustsBean.getPage_count() == 1) ? paramIllustsBean.getImage_urls().getMedium() : ((MetaPagesBean)paramIllustsBean.getMeta_pages().get(paramInt)).getImage_urls().getMedium();
  }
```

通过名称能够看出是一个获取url的函数，其中第一个参数是一个对象，这个对象用来记录图片的各种属性。对此次有帮助的就是`.getImage_urls().getMedium()` 

找到了要HOOK的目标函数，接下来就进入下一个阶段。



# HOOK

---

终于到了Frida大显神通的时候了！

## 搭建环境

因为整个过程中需要root，但是我自己主力手机并不准备root，所以决定使用安卓模拟器。

搭建过程参考自[Frida详细安装教程 - 简书 (jianshu.com)](https://www.jianshu.com/p/c349471bdef7)

### 模拟器

我一直使用的是夜神模拟器，因为其自带root，甚至安装xposed框架也很方便。是个极好的实验平台。后面将会使用adb链接它。

[夜神安卓模拟器-安卓模拟器电脑版下载-官网 (yeshen.com)](https://www.yeshen.com/)

### adb链接模拟器

通过adb能够链接安卓系统。众所周知adb有两种链接模式，一种是通过usb链接，另一种通过网络链接。如果要链接模拟器的话，使用网络比较方便。实际上我也不知道怎么通过usb链接模拟器。

[SDK Platform Tools 版本说明  | Android 开发者  | Android Developers (google.cn)](https://developer.android.google.cn/studio/releases/platform-tools)

下载得到adb之后，把他的路径加入环境变量，方便后面使用。

夜神模拟器在启动之后，默认会监听62001端口，这个就是adbd服务的端口，通过命令链接接好了

```Shell
adb connect 127.0.0.1:62001	# 链接ADB
adb devices			# 链接成功后 查看所有设备
adb shell			# 进入安卓系统的shell
```

当通过`adb devices`能够看到已连接的设备之后，就算链接成功了，之后将会通过adb传输文件到安卓系统

### 部署Frida

正主终于登场

#### 客户端

客户端使用Python开发，直接通过pip安装即可

```Shell
pip install frida
```

但是使用这个命令我会安装失败，所以我执行了

```Shell
pip install frida-tools
```

安装成功，执行命令

```Shell
frida --version
```

能看到版本就表示安装好了，非常简单

#### 服务端

[Releases · frida/frida (github.com)](https://github.com/frida/frida)

找到适合自己的版本即可，因为这次使用的是模拟器，所以cpu架构应该是x86的android，所以我下载了`frida-server-14.2.18-android-x86.xz` 。需要注意的是，一定要下载`frida-server`不能下载错了

最开始的时候，我想当然的下载了arm版本，结果不能运行，才想起来下载错了，然后下载了个x64的，结果不能运行，才注意到夜神模拟器是x86的android。

把下载下来的程序通过adb命令复制到安卓系统

```Shell
adb push frida /data/local/tmp
```

为什么是这个目录呢？因为别的目录不能执行程序，或者说不方便。

接着赋予可执行权限

```Shell
chmod +x ./frida
```

并且执行，程序没有任何输出，是正常的

![image-20210528161520012](https://qiniusave.5cser.com/mdimage-20210528161520012.png)

#### 链接Frida

执行两个命令实现端口转发，也即是在PC端开始监听，对端口接收到的数据，原封不动的交给安卓系统里面运行的 `frida服务端`

```Shell
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

此时，在cmd执行命令

```shell
frida-ps -U
```

应该能得到下图的结果 ceui.lisa.pixiv就是此次的目标

![image-20210528161601553](https://qiniusave.5cser.com/mdimage-20210528161601553.png)

## 编写脚本

首先贴上脚本再解释 

(代码修改自 [frida入门总结 - 『移动安全区』 - 吾爱破解 - LCG - LSG |安卓破解|病毒分析|www.52pojie.cn](https://www.52pojie.cn/thread-1128884-1-1.html))

(js部分参考[frida框架hook参数获取方法入参模板 - 小小咸鱼YwY - 博客园 (cnblogs.com)](https://www.cnblogs.com/pythonywy/p/13398295.html))

```python
import frida  #导入frida模块
import sys    #导入sys模块

jscode = """
    Java.perform(function(){  
        var MainActivity = Java.use('ceui.lisa.download.IllustDownload');   //获得MainActivity类
        MainActivity.getShowUrl.implementation = function(args, args2){     //Hook getShowUrl，用js自己实现
            send('Statr! Hook!' + args.getImage_urls().getMedium());          //发送信息，用于回调python中的函数
            return this.getShowUrl(args, args2);                            //原封不动的把函数还回去，只是拿到了参数
        }
    });
"""

def on_message(message,data): #js中执行send函数后要回调的函数
    print(message)

process = frida.get_remote_device().attach('ceui.lisa.pixiv') #得到设备并劫持进程com.example.testfrida（该开始用get_usb_device函数用来获取设备，但是一直报错找不到设备，改用get_remote_device函数即可解决这个问题）
script = process.create_script(jscode) #创建js脚本
script.on('message',on_message) #加载回调函数，也就是js中执行send函数规定要执行的python函数
script.load() #加载脚本
sys.stdin.read()
```

首先说明Frida的工作方式，所有的工作都是由服务端完成的，客户端只是用作控制。控制脚本使用Python编写，但是注入进去的代码却是由js编写的。我觉得选择使用js的原因可能是js是弱类型的脚本语言，对类型不敏感，对于上面的代码例子，完全不需要指定参数的类型就能使用这个对象。而即便是Python也不能实现这个操作，最起码得指定参数类型。

代码的注释已经清楚了

```python
process = frida.get_remote_device().attach('ceui.lisa.pixiv')
```

是对应的进程，名称可以由`frida-ps -U`得到

接下来把js脚本注册进去

再注册一个回调函数，这个函数对应js中的Send() ，第一次编写的时候，没反应过来这两个是对应的。不过说起来，回调函数的第二个参数是做什么的呢？



## 运行

首先把安卓程序运行起来，能够通过`frida-ps -U`得到进程名

然后运行Python脚本

最后，在安卓程序中操作，触发对应函数的调用即可。如下是运行结果，已经拿到了图片的url，很赞，已经开心的说不出话了。

![image-20210528161452351](https://qiniusave.5cser.com/mdimage-20210528161452351.png)



当然，这个只是Frida的小Demo，未来如果需要这个很厉害的软件的时候，会继续学习它。

## 但是！！！

我最开始的目的，并不是这个呀，，，一开始的目的是为了给某软件增加下载功能。。。

所以说，还需要别的技术，比如开发xposed插件，或者magisk插件。

不过Frida这个技术，对需要解密的程序非常重要，比如CTF比赛的时候，可能会用到这个技术。

