---
title: "破解一个极难的软件"
date: 2021-06-09T22:01:49+08:00
draft: false
tags: ["HOOK","Android","Xposed","反编译","frida","objection"]
categories: ["逆向分析"]
---

# 芜湖

喜大普奔，终于是成功破解这个软件。说是破解但是实际上这是个免费软件，但是软件不提供下载功能。以前的老版本软件是有下载功能的，但是没有收藏功能，很不好用，但是现在新版本有了收藏功能，但是没有了下载功能。。。并不知道是怎么想的，而且很多图片都没有了，好在我都给爬下来了~~

这次逆向过程非常不顺利，不过好在经过两个星期的空余时间，总算是解决掉了。现在记录一下整个过程。

有问题下方留言哈~

## 初次尝试

最开始的时候大概流程是先改安装包的后缀名，改成`.zip`然后解压出来，得到`.dex`文件，然后通过`dex2jar`工具转换成为`jar`包，接着通过`jd-gui`工具看到代码。

不过此时发现软件是加固的，想来也是，现在很多软件都会加固，所以我也了解一下常见的去壳方式。对于安卓来说，最稳妥的办法就是`dump`内存。以前脱壳的时候使用的是一个`xposed`模块，在虚拟机上面脱壳，我记得好像过程很顺利，但是最后得到的代码并不是很好看。看上去像是被混淆了一样。

考虑到加固技术发展的很快，我没有使用过去的老工具，直接去找了新的`dump`工具，`drizzleDumper`。通过`adb`传输到模拟器上面，运行软件，开启`drizzleDumper`，结果它突然发疯了，一直输出信息。首次使用这个工具，我也不知道是不是正常情况，所以我等了一会儿。觉得差不多了，就通过`Ctrl+C`终止了程序运行。此时`dump`了很多`.dex`文件，但是似乎只有两个大小。一个是`198kb`另一个是`2028kb`。

看上去可能是遇到反`dump`技术。但是不重要，既然已经得到了`dex`文件，就来看看里面都是什么吧。

![image-20210621215940455](https://qiniusave.xint.top/img/image-20210621215940455.png)

小的`dex`里面是这个，明显是`360`的加固，老朋友了，第一次破的壳就是`360`的，但是失败了。那是在两年前。现在就再次挑战它。

重要的代码显然不在这里，所以看另一个`dex`

![image-20210621220138398](https://qiniusave.xint.top/img/image-20210621220138398.png)

好家伙，看上去好像不太对。首先，并没有看到类似`入口`的代码，其次代码文件的命名全都变成`字母`了。另外有些代码并没有反编译出来。这个看上去就很棘手了。对于命名全都变成字母的问题，我一开始认为这个应该是代码经过了`混淆`。所以，也就继续去分析它的代码。

通过分析，发现软件用了`chromium`，可能是一个内建的浏览器？但是说实话，使用软件的过程并没有觉得它像是一个浏览器外壳。所以并不知道这个项目在这里的用处是什么。然后就是`android.support`这个包。此时看到这个包，我以为是安卓开发的公共代码。因为我在别的`apk`中也见过它，所以直接忽略了它。接下来，就是下面经过"混淆"的代码了。

好在虽然符号名被混淆了，但是内容基本上都反编译出来了。首先进行第一轮筛查，找出了代码文件比较大的。其次通过代码文件`import`的包，推测他是做什么的。最后筛查出来几个比较有价值的类(诸如`Js.class`)。

接着通过`frida`下钩子。结果`frida`一直提示找不到类，但是通过`jd-gui`看到类确实在那里。

接着翻阅`frida`的官方文档，找到了枚举所有类的方法，重定向到文件里面。确实发现了很多字母命名的类，并且也找到了对应的类。

![image-20210622222510434](https://qiniusave.xint.top/img/image-20210622222510434.png)

但是通过`frida`确确实实找不到类，此时就觉得很奇怪。后来留意了一下字母类的数量发现，`frida`只导出了`500+`个类，但是`jd-gui`却有足足`2500+`个类。此时觉得不对劲了。所以考虑是不是`dex`出了问题，于是考虑使用别的方法`dump`内存。

后来发现了`objection`工具，其中有个插件叫做[FRIDA-DEXDump](https://github.com/hluwa/FRIDA-DEXDump)

[objection - 基于frida的命令行hook工具食用手册 | Mario (strivemario.work)](http://strivemario.work/archives/8eec80c3.html)

使用这个工具得到了完全不同的结果

![image-20210622223049545](https://qiniusave.xint.top/img/image-20210622223049545.png)

看起来很靠谱的样子，使用新的反编译工具`jadx`工具，打开了`dex`文件，突然发现之前的操作都是白给。。。

![image-20210622223222755](https://qiniusave.xint.top/img/image-20210622223222755.png)

得到的代码非常完整，并且原本看到的字母命名的类已经全都没有了，此时突然醒悟过来。那个字母命名的类应该是反编译器确实找不到类的名字，然后自己给它命名了。通过`dex2jar`转成`jar`之后再用`jd-gui`打开，也是完整的代码。由此可见，这个锅必须由`drizzleDumper`来背。西内！

说起来，之前`drizzleDumper`也是个好伙计，只不过技术发展太快了，有名气的技术很快就会被针对。





## 思路

既然已经拿到了真正的代码，就可以开始正常的步骤了，接下来就是一通操作分析了。

基本思路就是先搭建好`frida`环境，为了方便，这次使用了`objection`工具，这样就能通过命令行快速验证目标函数是不是需要`hook`的了。如此一来肯定是很省时间了。通过`adb`连接夜神安卓模拟器调试软件，找到需要`hook`的关键函数，并且验证正确与否。

接着开发出相应的`xposed`模块，就可以开始修改软件了。

不过这地方有坑，两点

+ 夜神模拟器有两个版本，一个是`Android5`一个是`Android7`，其中前者自带`xposed`框架，但是后者没有。只有`root`权限，尝试给后者安装`xposed`结果失败了，索性不折腾了。决定最终模块运行在我手机的`vmos`虚拟机中。不过`frida`调试则还是在夜神模拟器中。
+ 连接`adb`的时候，建议找一个低版本的`adb`，否则会因为版本不匹配的关系不让使用`adb`，高版本可以连接`Android7`但是尝试连接`Android5`会失败。我使用的是`r24.0.4`版本。

## 配置基本环境

### 配置`frida`

这部分请看之前写的文章[使用Frida完成android的hook - 星途 · 小镇 (lonelysinging.github.io)](https://lonelysinging.github.io/posts/2021/05/28/frida_01/)或者看别的教程，此处不再赘述了

使用`adb`连接虚拟机，并且设置端口转发

```Shell
adb connect 127.0.0.1:62001
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

夜神模拟器`adb`端口默认是`62001`，如果是`Android5`的那个，好像是`52001`这个不确定，需要自行查询一下。

执行`frida-ps -U`命令能够正确获得进程列表表示`frida`已经配置完成了。

### 安装`objection`

这步可以参考这个博客[objection - 基于frida的命令行hook工具食用手册 | Mario (strivemario.work)](http://strivemario.work/archives/8eec80c3.html)写的比较好。如果英文水平足够，当然是直接建议去官网比较稳妥。

+ 安装

  `pip install objection` 

  安装时间非常长，不清楚是不是我网络的问题。另外，如果电脑同时安装了`python2`和`python3`的话，需要注意区分`pip`的版本，我用的是`python3`并且只安装了这一个。所以直接执行`pip`是没有问题的。

+ 指定被操作的进程

  `objection -g com.mmzztt.app explore`

+ 列出所有activity

  `android hooking list activities`

  ```java
  com.apicloud.tencentads.SplashActivity
  com.qq.e.ads.ADActivity
  com.qq.e.ads.LandscapeADActivity
  com.qq.e.ads.PortraitADActivity
  com.qq.e.ads.RewardvideoLandscapeADActivity
  com.qq.e.ads.RewardvideoPortraitADActivity
  // 上面的似乎都是和广告相关的东西
  com.uzmap.pkg.EntranceActivity	// 看起来是什么入口？
  com.uzmap.pkg.LauncherUI	// 似乎是入口相关的
  ```

  这里看`com.uzmap`前缀可能有什么戏

见到上面的结果，就证明`frida`和`objection`都已经安装完成了

### `dump`出dex

虽然前面说了怎么`dump`出dex，但是还是补充上如何操作。

+ 安装

  `pip install frida-dexdump`

+ 在虚拟机里面运行目标程序

+ 命令行直接运行`frida-dexdump`等待结束

  ![image-20210614210601035](https://qiniusave.xint.top/img/image-20210614210601035.png)

  好像不是很成功的样子，出现了一些错误。但是好像不是很重要。

+ 此时根据上图结果就能看到他把`dex`文件放到了用户目录下，使用`包名`命名的文件夹中了。

  ![image-20210622223049545](https://qiniusave.xint.top/img/image-20210622223049545.png)

  看起来比较顺利

### jadx

这是个新鲜的工具，在这里下载[skylot/jadx: Dex to Java decompiler (github.com)](https://github.com/skylot/jadx)

这个通过这个软件可以直接打开`dex`文件，不需要通过`dex2jar`工具转换了。真的是非常省事儿~

## 分析代码

分析代码之前，需要先想清楚需求是什么。此次想要修改的软件是一个看图的软件，但是我不准备直接给出软件名称，如果想知道是什么软件，请聪明的读者思考一下安卓包名的命名规则是什么~

这个软件就是一个看写真图片的软件，现在我想修改软件，使之可以实现双击下载功能。并且能够关闭掉讨厌的更新提示。

现在明确了目标，所以接下来需要关注的部分就是软件下载图片的部分，以及双击会执行的代码。至于自动更新，一开始是没啥头绪的，软件的入口并不是很明确，所以我不准备按照执行流程分析了。

### 相关目录分析

我一般在具体分析软件之前，会先去看看它的缓存目录，或者配置文件什么的地方，有时候能够发现非常有用的线索。如何发现相关目录呢？一般来说，重要的目录就是`Android/data/包名/`这样的目录，如果目标软件在`/sdcard/`下面也建立的文件夹的话，也可以看看。如果有`root`权限的话，可以去软件的`/data/data/包名`底下看看，有时候可以看到配置文件什么的。能够提供相关线索。

而通过`objection`很方便就能得到相关目录。啥都有了，此时通过`ES文件浏览器`什么的去转一圈，总能发现好玩的。

![image-20210624222217118](https://qiniusave.xint.top/img/image-20210624222217118.png)

尤其是这次，直接找到了软件缓存文件的地方，就是`Android/data/包名/`下

![image-20210624221641554](https://qiniusave.xint.top/img/image-20210624221641554.png)

看到这个文件名和大小，就已经瞬间明白这个是什么了。通过图片的方式打开，果不其然就是缓存的图片。而其文件的命名，不用看也能知道是`32`位的`md5`。对这个东西已经太熟悉了。

所以，接下来，就不需要考虑自己下载了，只需要通过代码找到操作缓存文件的地方即可，然后直接复制缓存文件，重命名即可得到"下载"的文件。

### 找有价值的函数

![image-20210624220518576](https://qiniusave.xint.top/img/image-20210624220518576.png)

没什么好说的，基本上反编译工具已经把类名和函数名反编译过来了，所以基本上把大概代码浏览一遍，发现这样明显的类了，接下来主要是验证类和函数了。基本思路就是通过`objection`对整个类的所有函数都下`hook`，然后再操作软件，看看会调用到哪个函数。确定函数价值。

例如通过以下命令操作

```Shell
# 列出类的所有函数
android hooking list class_methods com.uzmap.pkg.uzmodules.photoBrowserSu.PhotoBrowser

# hook方法，列出其参数和返回值，还有堆栈
android hooking watch class_method com.uzmap.pkg.uzmodules.photoBrowserSu.ImageBrowserAdapter.$init --dump-args --dump-backtrace --dump-return

# hook整个类的所有方法
android hooking watch class android.support.v4.view.PagerAdapter
```

列出类的所有函数，对比和`jadx`的结果，确保找到的类和方法是正确的，别像我一样傻乎乎的对着一堆不存在的方法下`hook`。

`hook`方法的话，则能看到方法的返回值、参数、以及调用堆栈。前面两者肯定不用考虑非常重要，而对于堆栈，如果是想要分析软件类和方法的调用关系，则是非常好用的信息。能够帮你理清谁调用谁，通过这些信息，也能进一步推测反编译不出类名的类是干什么的。如果想要实现复杂的功能，可能确实需要明白目标软件是怎么工作的。

而`hook`整个类的话，看不到方法的参数和返回值。但是能够帮忙确定类中哪个方法被调用了，也是非常重要的。能够一次性筛查出重要的类。

### 获取url列表

经过一番查找，发现软件工作原理是有一个缓存管理类。需要这张图片的话，就会问这个缓存管理类要，如果已经有缓存的图的话，直接返回路径，然后载入。否则的话，才会下载。但是经过简单看代码之后，并没有发现一个保存取`md5`文件名的数组，但是发现了这个

![image-20210624230628010](https://qiniusave.xint.top/img/image-20210624230628010.png)

这个构造函数有一个参数是一个`ArrayList`通过`hook`这个函数，能够发现里面存的是原始的`url`。比较麻烦，但是咱们上面已经发现了文件名就是什么东西取了`md5`，是不是就是`url`取`md5`之后直接做文件名呢？

`hook`函数之后，可以直接看到参数，对于`java`自带的类型，`objection`可以直接看到其内容。拿到第一个元素，取`md5`之后，对比其缓存的文件，确实发现了同名文件。所以，只需要对这个数组中的元素取`md5`就能在缓存目录找到对应的文件，之后复制走，加个后缀就能作为正常图片打开了。

### 确定当前图片索引

接下来就是要知道现在看的是哪张图片，首先引入眼帘的就是浏览界面的进度。但是该怎么找到它呢？

![image-20210624231324273](https://qiniusave.xint.top/img/image-20210624231324273.png)

最快速的办法，就是通过布局文件找。

通过`apktool`解包`apk`文件，之后得到了布局文件

![image-20210624231515528](https://qiniusave.xint.top/img/image-20210624231515528.png)

结果如您所见，似乎并没有相关的布局数据。所以，考虑这部分可能是动态产生的布局。所以，没办法，还是得继续找代码。其实如果顺利的话，通过布局文件里面的`id`属性，能够直接找到对应的类，这样对这个类中的方法下`hook`就能直接获取到当前图片的索引了。

依旧是上面的老办法，继续对可疑的类下钩子。最后发现了一个有用的方法

![image-20210624231855129](https://qiniusave.xint.top/img/image-20210624231855129.png)

从名字上看，就能知道这个可能是用来设定当前看的这张图的索引的。第二个参数`position`，应该就是当前图片索引。对这个函数下断点

![image-20210624232142901](https://qiniusave.xint.top/img/image-20210624232142901.png)

能够看到这个参数就是当前看到的图片的索引减一。那么它的值应该对应的就是数组的下标了。

## 找到双击操作

既然收集到了需要的数据，接下来只需要找到一个触发函数，把相关的复制文件的逻辑放进去，就完成了这个目标。找双击函数的思路，就是看`photobrower`这个类，因为软件可以双击放大。

![image-20210624232855877](https://qiniusave.xint.top/img/image-20210624232855877.png)

找到了这个函数`onDoubleTap()`，但是，这个函数是一个匿名对象里面方法。并且我没有找到对应的`hook`方法，所以只能走偏方了。我注意到它几乎一定会执行`smoothScale()`函数，我只需要`hook`它就行了。

至此，就通过`objection`找到了所有需要的函数，于是就可以开始写`xposed`模块了。

## 升级功能的屏蔽

对于这个，通过字符串搜索，确实找到了对应的配置类。但是下的钩子并没有被触发，因为软件是平台生成的，所以代码不被使用也很正常。后面分析发现，更新相关的方法名反编译失败了。也就是说，即使找到了对应的逻辑也不能正确的`hook`，这样的话就没有意义了。

![image-20210624233906344](https://qiniusave.xint.top/img/image-20210624233906344.png)

但是也并不是没有办法了，观察这个升级框，点击返回什么的都不能关闭它，只能点击立即更新，之后就能继续使用软件了。但是会后台下载安装包，之后更新。不过在下载的时候也可以正常使用软件。所以，另一个思路就是让这个弹窗弹不出来。

![image-20210624233935129](https://qiniusave.xint.top/img/image-20210624233935129.png)

通过分析布局文件，依旧是没找到对应的布局，也就找不到对应的类了。这样的话，只能通过别的方法了。考虑到动态的布局，最终也会是一个布局文件。所以，是否有办法把运行的界面布局`dump`出来呢？其实这个是最近使用`Autojs`软件的时候知道的功能。去百度了如何`dump`布局，果然通过`adb`是可以`dump`布局的。(实际上，`Android Studio`是有这样的功能的，但是一直失败，不清楚是为什么)

下面两个命令需要进去`adb shell`

```shell
uiautomator dump /sdcard/ui.xml
# 获取顶层的布局

dumpsys window | grep  mCurrentFocus
# 获取顶层的Activity
```

后者只有一开始发现的`com.uzmap.pkg.LauncherUI` 显然也是没什么用的。但是前者，其中发现了一些`id`，

![image-20210624234943313](https://qiniusave.xint.top/img/image-20210624234943313.png)

众所周知，这个`id`是注册在`public.xml`这样的文件中的，找到了如下图这样的东西。后面的`id`是十六进制，转换成十进制，然后搜索`R.java`文件，就能找到具体是那里使用了这个布局。

![image-20210624235036862](https://qiniusave.xint.top/img/image-20210624235036862.png)

最终发现，直接搜索`id`就能直接找到代码。。。转换进制什么的完全没有必要。。。

![image-20210624235433486](https://qiniusave.xint.top/img/image-20210624235433486.png)

然后最终在附近的类中看到了下面这个方法

![image-20210620113953432](https://qiniusave.xint.top/img/image-20210620113953432.png)

`hook`了之后，确定了这个就是弹出更新弹窗的方法。接下来，就可以开始正式开发`xposed`模块了



## xposed模块

直接贴出代码，若是对`java`没有一些了解，大概完全不用考虑学习编写模块了。但是如果会`java`的话，下面的代码应该能够直接看懂。

对于`xposed`框架代码的编写，主要就是用到了`java`的反射。我理解的反射就是把类的信息给映射成一个对象，通过这个对象可以得到对应的类信息。由此，可以得到类的结构。

另外，在比较重要的一点就是，如果代码是加固的，那就稍微麻烦一点。需要拿到加固的类加载器，具体看这里[使用xposed来hook使用360加固的应用 - 简书 (jianshu.com)](https://www.jianshu.com/p/8324ad8860d2?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

还有，就是在手机上的`vmos`虚拟机，`xposed`的日志不能正常工作，所以我写了一个`writefile()`方法，用来做日志输出。

除此之外，没有什么值得注意的了。

### 代码

```Java
package com.example.startu.myapplication233;

import android.app.Application;
import android.content.Context;
import android.os.Environment;
import android.view.View;
import android.widget.Toast;

import java.io.File;
import java.io.FileOutputStream;
import java.io.UnsupportedEncodingException;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.ArrayList;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XC_MethodReplacement;
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
	// 这地方没有去找软件里面的方法，觉得太麻烦了。所以直接去网络上找到代码复制过来了
    private static String getMD5(String info) {
        try {
            //获取 MessageDigest 对象，参数为 MD5 字符串，表示这是一个 MD5 算法（其他还有 SHA1 算法等）：
            MessageDigest md5 = MessageDigest.getInstance("MD5");
            //update(byte[])方法，输入原数据
            //类似StringBuilder对象的append()方法，追加模式，属于一个累计更改的过程
            md5.update(info.getBytes("UTF-8"));
            //digest()被调用后,MessageDigest对象就被重置，即不能连续再次调用该方法计算原数据的MD5值。可以手动调用reset()方法重置输入源。
            //digest()返回值16位长度的哈希值，由byte[]承接
            byte[] md5Array = md5.digest();
            //byte[]通常我们会转化为十六进制的32位长度的字符串来使用,本文会介绍三种常用的转换方法
            return bytesToHex1(md5Array);
        } catch (NoSuchAlgorithmException e) {
            return "";
        } catch (UnsupportedEncodingException e) {
            return "";
        }
    }

    private static String bytesToHex1(byte[] md5Array) {
        StringBuilder strBuilder = new StringBuilder();
        for (int i = 0; i < md5Array.length; i++) {
            int temp = 0xff & md5Array[i];//TODO:此处为什么添加 0xff & ？
            String hexString = Integer.toHexString(temp);
            if (hexString.length() == 1) {//如果是十六进制的0f，默认只显示f，此时要补上0
                strBuilder.append("0").append(hexString);
            } else {
                strBuilder.append(hexString);
            }
        }
        return strBuilder.toString();
    }

   	// 依旧不想使用软件里面的复制方法，直接复制网上的代码了。但是可能在Android11上面出现问题，具体看下面说明
    public void copyfile(File fromFile, File toFile,Boolean rewrite ) throws Throwable
    {
        if (!fromFile.exists()) {
            return;
        }
        if (!fromFile.isFile()) {
            return ;
        }
        if (!fromFile.canRead()) {
            return ;
        }
        if (!toFile.getParentFile().exists()) {
            toFile.getParentFile().mkdirs();
        }
        if (toFile.exists() && rewrite) {
            toFile.delete();
        }

        try {
            java.io.FileInputStream fosfrom = new java.io.FileInputStream(fromFile);
            java.io.FileOutputStream fosto = new FileOutputStream(toFile);
            byte bt[] = new byte[1024];
            int c;
            while ((c = fosfrom.read(bt)) > 0) {
                fosto.write(bt, 0, c); //将内容写到新文件当中
            }
            fosfrom.close();
            fosto.close();

        } catch (Exception ex) {
            writefile(ex.toString());
        }
    }

    public void handleLoadPackage(final LoadPackageParam loadPackageParam) throws Throwable {
        XposedBridge.log("Loaded app: " + loadPackageParam.packageName);
        // writefile("启动包: "+loadPackageParam.packageName);
        if (loadPackageParam.packageName.equals("com.mmzztt.app")) {
            writefile("发现目标: "+loadPackageParam.packageName);
            Class<?> clazz = XposedHelpers.findClass("com.stub.StubApp", loadPackageParam.classLoader);
            // Class clazz = loadPackageParam.classLoader.loadClass("ceui.lisa.download.IllustDownload");
            XposedHelpers.findAndHookMethod(clazz, "interface7",Application.class,Context.class, new XC_MethodHook() {
                protected void beforeHookedMethod(XC_MethodHook.MethodHookParam param) throws Throwable {
                    super.beforeHookedMethod(param);
                    writefile("进入函数: ");
                    Context context = (Context) param.args[1];
                    //获取classloader，之后hook加固后的就使用这个classloader
                    ClassLoader realClassLoader = context.getClassLoader();
                    hookCheckoutXposed(realClassLoader,loadPackageParam);
                    writefile("hook结束");
                }

                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    // param.setResult("你已被劫持");
                    // writefile("新的图片:");
                }
            });
        }
    }

    private ArrayList<String> mImagePaths;
    private int pos;
    private  String cache_files="/sdcard/Android/data/com.mmzztt.app/cache/UIPhotoViewer/";
    private String mmttzz_files = "/sdcard/mmzztt/";
    private boolean is_copy = false;
    Context context;

    public void hookCheckoutXposed(ClassLoader classLoader,final LoadPackageParam loadPackageParam) throws Throwable {
        writefile("进入hook函数");

        Class Context_c = loadPackageParam.classLoader.loadClass("android.content.Context");

        // 获取 Context
        XposedHelpers.findAndHookMethod("com.uzmap.pkg.uzcore.uzmodule.UZModule", classLoader, "initPlatform", Context_c, new XC_MethodHook() {
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                if(param.args[0] != null) {
                    context = (Context)param.args[0];
                    cache_files = context.getExternalCacheDir().getAbsolutePath() + "/UIPhotoViewer/";
                    writefile("cache_files: "+cache_files);
                }
            }
        });

        XposedHelpers.findAndHookMethod("android.support.v4.view.PagerAdapter", classLoader, "setPrimaryItem", View.class,int.class,Object.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                if(param.args[1] == null){
                    pos = 0;
                }else{
                    pos = (int)param.args[1];
                }
                is_copy = false;

            }
        });

        XposedHelpers.findAndHookMethod("com.uzmap.pkg.uzmodules.photoBrowserSu.view.largeImage.LargeImageView", classLoader, "smoothScale", float.class,int.class,int.class, new XC_MethodHook() {
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                writefile("双击了");

                String url = mImagePaths.get(pos);
                if(url == null){
                    Toast.makeText(context,"获取图片url失败",Toast.LENGTH_SHORT).show();
                    return ;
                }
                String md5_string = getMD5(url);
                String md5_name = md5_string+".jpg";
                String file_name = md5_name;

                writefile("定位"+pos+", 内容: " + url + ", md5: "+md5_string);

                String[] l_list = url.split("/");
                if(l_list != null){
                    file_name = l_list[l_list.length-1];
                }
                writefile("file_name: "+file_name+", l_list.length: "+l_list.length);
                if(!is_copy) {
                    copyfile(new File(cache_files + md5_string), new File(mmttzz_files + file_name), true);
                    Toast.makeText(context,"保存图片"+file_name,Toast.LENGTH_SHORT).show();
                    is_copy = true;
                }
            }
        });

        Class UZModuleContext_c = loadPackageParam.classLoader.loadClass("com.uzmap.pkg.uzcore.uzmodule.UZModuleContext");
        XposedHelpers.findAndHookMethod("com.apicloud.dialogBox.DialogBox", classLoader, "jsmethod_alert",UZModuleContext_c, new XC_MethodReplacement() {
            @Override
            protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
                writefile("禁止一个 alert弹窗");
                Toast.makeText(context,"你是真的想更新嘛？",Toast.LENGTH_SHORT).show();
                return null;
            }
        });

        Class ImageLoader_c = loadPackageParam.classLoader.loadClass("com.uzmap.pkg.uzmodules.photoBrowserSu.ImageLoader");
        Class ImageBrowserAdapter_c = loadPackageParam.classLoader.loadClass("com.uzmap.pkg.uzmodules.photoBrowserSu.ImageBrowserAdapter");

        XposedHelpers.findAndHookConstructor(ImageBrowserAdapter_c, Context_c,UZModuleContext_c,ArrayList.class,ImageLoader_c,String.class,  new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                writefile("进入 ImageBrowserAdapter 的构造函数了");
                try {
                    mImagePaths = (ArrayList<String>) param.args[2];
//                    for(int i=0;i<mImagePaths.size();i++){
//                        writefile(mImagePaths.get(i));
//                    }
                    writefile("数组长度: " + mImagePaths.size());
                }catch (Exception e){
                    writefile(e.toString());
                }
            }
        });
    }
}

```

### 弹出toast

这部分需要获取目标软件的`Context`，这个需要分析代码，一般来说，没有加固的代码，在入口类中就能找到。但是如果是加固之后的，可能需要找加固方案的代码了。具体看上面的代码，找到什么方法参数是`Context`的就能获取到，当然是不是任意一个`Context`都行我就不清楚了。

### 关于复制文件

手机更新`Android 11`之后，对应用就不能随意访问`Adnroid/data/`目录了，所以上面的代码可能会复制文件失败，因为模块并不是目标软件，所以可能会有权限问题，当然只是猜测。遇到这种情况，可以试试`hook`目标软件自己的复制文件方法。或者拿到`url`自己下载，或者直接拿到输出流写到自己的目录。



## 操作过程中可能需要的操作

### 启用ADB网络调试

```Shell
TCP/IP方式：
setprop service.adb.tcp.port 5555
stop adbd
start adbd
 
usb方式：
setprop service.adb.tcp.port -1
stop adbd
start adbd
```

from:[(2条消息) Android 网络调试 adb tcpip 开启方法_Shawn Kong的专栏-CSDN博客](https://blog.csdn.net/shawnkong/article/details/8923933)

### 禁用 SELinux

```Shell
setenforce 0
```

- 级别0：表示Permissive模式。
- 级别1：表示Enforcing模式。
- 至于disabled模式和其他模式的切换只能修改配置文件,命令不起作用。其次,修改完成之后,必须重启系统才能够生效。



from: [Android查看SELinux状态及关闭SELinux - 简书 (jianshu.com)](https://www.jianshu.com/p/27b69ba08aac)

### apktool

#### 解包

```Shell
apktool.jar d Shaft_3.1.5.apk -o Sha
```

打包

```Shell
apktool.bat b [资源文件夹] [打包生成的apk文件] 
```

from:[(2条消息) 使用ApkTool_一起进步的博客-CSDN博客_apktool](https://blog.csdn.net/qq_27292113/article/details/79931268)

### BusyBox

[meefik/busybox: BusyBox for Android (github.com)](https://github.com/meefik/busybox)

### 添加库

```java
repositories {
    jcenter()
}

compileOnly 'de.robv.android.xposed:api:82'
compileOnly 'de.robv.android.xposed:api:82:sources'
```



### 新项目需要添加的

```xml
        <meta-data
            android:name="xposedmodule"
            android:value="true" />
        <meta-data
            android:name="xposeddescription"
            android:value="模块描述" />
        <meta-data
            android:name="xposedminversion"
            android:value="54" />
```



### 旧版本下载地址

[Android Studio 下载文件归档  | Android 开发者  | Android Developers (google.cn)](https://developer.android.google.cn/studio/archive)

3.2.1版本

### 遇到的问题


使用旧版本 初始化项目的时候就会出错

[Unable to resolve dependency for ‘:app@debug/compileClasspath‘: Could not resolve com.android.suppor_达帮主-CSDN博客](https://blog.csdn.net/qq_35427437/article/details/79835673)

## 操作过程的一些记录

+ 安装

  `pip install frida-dexdump`

+ dump dex

  `frida-dexdump`

  

+ 通过`jafx-gui`打开dex

  ![image-20210614212913273](https://qiniusave.xint.top/img/image-20210614212913273.png)

  绝了，这个软件的反反编译能力我觉得挺强的

+ 终于找到了关键方法

  `com.uzmap.pkg.uzmodules.photoBrowserSu.ImageDownLoader.saveFile(java.io.InputStream, java.lang.String)`

+ 通过分析xml，找到了菜单

+ 找到了取消按钮的点击事件

  ![image-20210614233042906](https://qiniusave.xint.top/img/image-20210614233042906.png)

  但是通过这个方法并不能得到什么有用的信息，好像拿不到需要的图片。所以考虑别的方法。

  我有注意到似乎有双击操作。。。
  
+ 使用前面的xposed模板，写入类名和方法，一次成功

+ [Xposed第一课(微信篇) hook含有多个参数的方法 - 简书 (jianshu.com)](https://www.jianshu.com/p/c730b5e0c150)

+ 通过xposed成功拿到双击会触发的方法，感觉快看到胜利的曙光了

+ ![image-20210615233012378](https://qiniusave.xint.top/img/image-20210615233012378.png)

+ 上一步找到了双击，接下来就要获取到当前正在看的图片了

  ![image-20210616230425945](https://qiniusave.xint.top/img/image-20210616230425945.png)

  此处找到了在载入图片完成之后会调用的方法了，但是这个方法并不好用

  于是找到了更好用的办法

  android.support.v4.view.ViewPager;

  ![image-20210616235043719](https://qiniusave.xint.top/img/image-20210616235043719.png)

  这两个是加载和预加载，通过这两个可以得到加载的具体的具体图片

+ ImageBrowserAdapter 构造函数可以获得列表，但是其中都是url需要取md5

+ `uiautomator dump /sdcard/ui.xml` 命令可以得到当前最顶层的布局

+ `dumpsys window | grep  mCurrentFocus` 命令可以得到最顶层的`Activity`

+ 禁止弹窗

  

