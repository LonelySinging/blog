---
title: "Xposed初体验"
date: 2021-05-31T19:33:48+08:00
tags: ["HOOK","Android","Xposed","反编译"]
categories: ["逆向分析"]
---



# Xposed

对于`Xposed`这个框架，在很早之前就已经了解到了，那时候是用它来实现自动抢红包这样的功能，总之是一个非常厉害的框架。不过这些年似乎热度正逐渐被`Magisk`夺取，不过因为两者实现方式不同，所以在不同的场景下还是各有各的存在意义。比如说`Magisk`使用起来风险更低，因为它并没有真正修改系统文件，也就不会轻易出现系统崩溃，卡开机屏的问题。不过要说插件数量，据我所知还是老牌的`Xposed`比较多。

`Xposed`是通过劫持安卓系统最初始的进程`zygote`实现工作的，任意一个进程总是通过`fork zygote`产生的，所以，`Xposed`劫持了它之后，就能对其他进程做任何事情了。

因为目前并不打算`root`手中的主力机，所以打算使用一部不用的手机`金立s5`来作为试验机，不过在我辛苦的`root`它之后，发现想要安装`Xposed`还真是麻烦。并且我也不想维护两个手机，并且我知道手机上会有虚拟机，它一般都会自带`Xposed`框架。由此，觉得`Xposed`会更方便。说起来，如果在主力机上面实现了一些实用的功能，使用起来也会很方便。

我也尝试过安装`Magisk`，但是，并没有成功，在通过`fastboot`刷写`boot`的时候电脑一直找不到设备，不清楚是驱动问题，还是手机限制的问题，而且这部手机网上能找到的资料很少，所以不方便折腾。于是就完全放弃了使用`s5`作为试验机的想法。



# 环境搭建

目前手持主力机`红米K30U`，联发科的SOC使得这个手机像是后娘养的一样，啥东西都赶不上热乎的，希望未来随着滥发可机型变多小米能够对其有更多的优化。不过好在并不影响我的使用。

现在的设想是通过安装`VMOS`这个虚拟机软件在手机上面运行一个系统，免费版本的是`Android 5.1.1`，里面带了`ROOT`还有`Xposed`并且能够很方便的打开`网络adb`方便使用笔记本对其操作，实际上这也是安装软件和传输文件最方便的方式了，尤其是能够和`Android Studio`无缝协作。

## 配置开发环境

### 安装Android Studio (AS)

下载链接在此https://developer.android.google.cn/studio/

此时有需要说明的是，网络上讲关于`Xposed`模块开发的教程都会提到关闭`AS`的`Instant Run`，但是我下载最新版本之后，在设置中并没有找到这个的设置项。让我困惑了很久，直到无意间了解到这个功能在`3.5`版本的时候就被谷歌"优化"掉了。被新的`Alpply Change`功能替代了。但是不清楚这个新的功能是否会导致出现教程中说到的异常，我是用的是`Android Studio 3.2.1`好在官网能够下载到。

安装之后，国内可能需要设置代理才能够愉快的下载`SDK`以及相关`jar`依赖库，毕竟是谷歌的服务，被墙了也是毫不意外的事情。

之后，需要下载`SDK`，建议下载`7.1.1 (API 25)`、`5.1.1 (API 22)`，这两个是最多的版本了，另外，对于`Android Studio 3.2.1`来说，`SDK 10 (API 29)`似乎是必须安装的，对于`Android Studio 4.2`来说，必须安装`SDK 11 (API 30)`，总共下载大小似乎还没有超过一个G。所以梯子流量不多的人也不用担心。

### 创建项目

再然后，创建一个项目，可以是没有`MainActivity`的项目，不过我选择了一个`Empty MainActivity`的项目，目的是为了好看 : P  (实际上是为了方便观察`apk`是否真的安装成功了)

### 添加新的

需要在项目的`AndroidManifest.xml`中添加 (具体参看网上很多的教程，这部分是没有问题的)

```xml
<meta-data
    android:name="xposedmodule"
    android:value="true" />
<meta-data
    android:name="xposeddescription"
    android:value="我是一个Xposed例程" />
<meta-data
    android:name="xposedminversion"
    android:value="53" />
```

添加这三个条目的原因是为了能让`Xposed Manager`认识开发的模块。此时编译安装之后，`Xposeed Manger`就能看到这个模块了

![image-20210606141722656](https://qiniusave.5cser.com/img/image-20210606141722656.png)

需要说明的是，如果一开始创建项目的时候没有`Activity`，格式可能和我图片中的不一样，所以这也是我创建了一个带`Activity`的项目

### 导入模块所依赖的`jar`包

需要添加以下内容

```json
repositories {
    jcenter()
}
provided  'de.robv.android.xposed:api:82'
provided  'de.robv.android.xposed:api:82:sources'
```

![image-20210606141342040](https://qiniusave.5cser.com/img/image-20210606141342040.png)

添加完毕之后，`Android studio`会提示你`Sync`，点击之后，就会自动从仓库下载对应的`jar`到项目中，很是方便。(但愿你开了代理)

> *provided 的作用是 在打包apk时不包含依赖包* 
>
> 一开始自作聪明的认为`82`指的是版本号，我安装的是`89`版本，所以修改成了`89`结果失败了，真是自作聪明
>
> [Xposed模块开发教程(二) 第一个Xposed模块应用-在手机状态栏增加显示cpu温度_没错, 我是东芝-CSDN博客_com.android.systemui.statusbar.policy.clock](https://blog.csdn.net/u014418171/article/details/52911715)

### 创建用于`hook`的代码

新建代码文件，此处我在`入口类同级目录`创建了一个类文件，取名`HookTest`，它是继承于`IXposedHookLoadPackage`接口的，这部分代码和可以从网上教程中复制(指导入的库名称以及`IXposedHookLoadPackage`这个很长的接口名称)

![image-20210606141916216](https://qiniusave.5cser.com/img/image-20210606141916216.png)

上图的最终代码我会提供到最后，并且做出解释

## 开始编写代码

对于此次任务，依旧是获取`Shaft`这个软件的图片链接，对于这部分内容直接参看之前的文章

[使用Frida完成android的hook - 星途 · 小镇 (lonelysinging.github.io)](https://lonelysinging.github.io/posts/2021/05/28/frida_01/)

通过反编译，得到我们需要`hook`的方法，当然，如果有源码当然是最好的了，不过如果是这样的话，直接修改源码岂不是更快？

现在假设你根据上面的文章得到了需要`hook`的方法

### 代码

那么我的代码是下面这样的

```Java
package com.example.startu.myapplication233;

import android.os.Environment;

import java.io.File;
import java.io.FileOutputStream;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

public class HookTest implements IXposedHookLoadPackage {

    public void writefile(String str) throws Throwable{
        String sdCardDir =Environment.getExternalStorageDirectory().getAbsolutePath();
        File saveFile = new File(sdCardDir, "aaaa.txt");
        FileOutputStream outStream = new FileOutputStream(saveFile,true);
        outStream.write((str+"\n").getBytes());
        outStream.close();
    }

    public void handleLoadPackage(final LoadPackageParam loadPackageParam) throws Throwable {
        XposedBridge.log("Loaded app: " + loadPackageParam.packageName);
        writefile("启动包: "+loadPackageParam.packageName);
        if (loadPackageParam.packageName.equals("ceui.lisa.pixiv")) {
            writefile("发现目标: "+loadPackageParam.packageName);
            Class<?> clazz = XposedHelpers.findClass("ceui.lisa.models.ImageUrlsBean", loadPackageParam.classLoader);
            XposedHelpers.findAndHookMethod(clazz, "getMedium", new XC_MethodHook() {
                protected void beforeHookedMethod(XC_MethodHook.MethodHookParam param) throws Throwable {
                    super.beforeHookedMethod(param);
                }

                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    // param.setResult("你已被劫持");
                    writefile("新的图片:");
                    writefile((String)param.getResult()+"\n");
                }
            });
        }
    }
}
```

### 解释

作为`Xposed`的模块，当然是以回调的方式被调用。而需要编写的回调函数就是`handleLoadPackage`函数，这个函数将会在每次系统创建进程的时候被调用。通过`if (loadPackageParam.packageName.equals("ceui.lisa.pixiv"))`过找到需要`hook`的进程，接着找到类和方法。实际上如果会`Java`的话，对于上面的代码应该一眼就看懂了。唯一需要补充的是，`beforeHookedMethod`和`afterHookedMethod`这两个方法，他们分别在执行原函数之前和执行原函数之后被调用，前者你可以修改其参数，后者可以修改其返回值。这样会让原函数很没有尊严。。。



> 另外需要解释，我为什么会写一个`writefile`函数，为什么不用框架自带的`log`系统。原因是因为我发现在手机安装的虚拟机中，`Xposed`的日志系统会失效，显示不出来日志。我曾一度以为模块没有正常工作，为此我换了各种虚拟机，各种版本。折腾了一下午，最后想到可能是日志系统出问题了？最后抱着试一试的心态写了一个`writefile`函数，没想到工作正常，当时就气哭了 (
>
> 实际上，框架在`夜神模拟器 (Android 5版本)`上工作完全正常，不过它的`Android 7`版本并没有自带`Xposed`框架，应用商场的是`Android 5`的，如果安装了会导致卡开机屏幕

### 指定入口类

`Xposed`模块在会从入口配置`xposed_init`文件里找入口类，在`assets`创建一个`xposed_init`文件，如果没有`assets`文件夹，则自行创建

![image-20210606145459689](https://qiniusave.5cser.com/img/image-20210606145459689.png)

其内容是上面编写的`Hook`类



## 结果

![image-20210606150735724](https://qiniusave.5cser.com/img/image-20210606150735724.png)

在`储存目录`下能看到命名为`aaaa.txt`的文件，其中有所有启动的进程，以及`Shaft`得到的所有图片的链接

通过`Xposed`和`Frida`都能实现这样的功能，但是也能看出他们的区别，一般来说`Frida`需要电脑配合(通过手机安装`Termux`之类的软件应该可以免电脑)，但是`Xposed`则不需要，而且适合长期修改。`Frida`方便用于解密，或者用于破解认证一类不需要长久`hook`的需要。



**参考 无敌的网友**

[Xposed模块开发教程整理 - leigq - 博客园 (cnblogs.com)](https://www.cnblogs.com/leigq/p/13406599.html#)

[【Xposed模块开发入门】真·第一课 - 『移动安全区』 - 吾爱破解 - LCG - LSG |安卓破解|病毒分析|www.52pojie.cn](https://www.52pojie.cn/thread-688466-1-1.html)

