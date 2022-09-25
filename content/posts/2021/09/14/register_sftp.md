---
title: "Kconnect使用sftp windows自定义协议"
date: 2021-09-14T08:29:14+08:00
draft: true
tags: ["折腾","Android","Kconnect","局域网","windows"]
categories: ["折腾"]
---



终于有时间写点东西了，上次写东西已经是三个月之前了。自从出现了觉得一个月写一篇文章也没关系的想法之后就已经完全忘记有这回事儿了。一直觉得没有足够的时间，但是又想写出质量比较好的文章，所以就一直没有动笔(键盘)。



## Kconnect 远程文件管理

最开始知道这个软件是在`manjaro`上面，是和`KDE`桌面属于一个软件套件的软件。当时只是简单的使用之后觉得用处并不大，尤其是里面很多的功能完全用不到，也就是文件传输，还有同步剪切板这两个功能对我来说比较有用。可是就发送文件这样的功能也会有问题，比如说电脑不能一次给手机发送多个文件。而且经常手机收不到文件，不过好在手机给电脑发送并没有问题。而且支持跨平台，电脑切换系统之后并不影响`kconnect`的连接。

[KDE Connect | KDE Connect: A project that enables all your devices to communicate with each other.](https://kdeconnect.kde.org/)

里面有一个叫做`远程文件系统浏览器`的插件，但是一直不知道怎么用。后来知道了是通过在手机开启一个`sftp`服务，电脑通过`sftp://`启动`sftp`客户端，然后来连接手机，实现文件系统的浏览。

但是问题来了，`kconnect`在点击浏览文件之后会弹出一个框

![image-20210916080729822](https://qiniusave.xint.top/mdimage-20210916080729822.png)

![image-20210916080751405](https://qiniusave.xint.top/mdimage-20210916080751405.png)

直观的第一个感觉就是软件打开了一个`sftp://`的链接，但是因为我的电脑没有注册`sftp://`协议，以至于没有对应的软件被打开。

但是我的电脑安装了`filezilla`这个软件的，所以首先尝试是否能够通过这个软件直接连接到手机上面的`sftp`端口，但是现在需要知道是哪个端口。众所周知，如果没有`root`权限，应用是监听不了`0-1024`端口的，所以，肯定不是`22`端口。

这里直接通过`zenmap`扫描了一波手机，知道监听的端口是`1794`。但是通过匿名的方式连接这个端口是连接不上的，也就是说，他是有密码的。但是密码是什么呢？

按照正常流程，如果想让电脑连接手机，那么手机就需要先把账号密码传递给电脑。那么我能不能抓包呢？答案是不行的，使用`火绒剑`能看到手机确实给电脑发送了数据，并且最终通过`IE`框架打开了什么东西，但是通过`wirshark`抓到的数据是通过`ssl`加密的。我真笨，当然不会明文传输密码了。那么传输的是什么东西呢？

我是用`kde`系列软件的一个原因之一就是因为它们开源，那么现在去找到安卓客户端的源码。

https://invent.kde.org/network/kdeconnect-android/

```Java
if (server.start(storageInfoList)) {
                if (preferences != null) {
                    preferences.registerOnSharedPreferenceChangeListener(this);
                }
                NetworkPacket np2 = new NetworkPacket(PACKET_TYPE_SFTP);
                //TODO: ip is not used on desktop any more remove both here and from desktop code when nobody ships 1.2.0
                np2.set("ip", server.getLocalIpAddress());
                np2.set("port", server.getPort());
                np2.set("user", SimpleSftpServer.USER);
                np2.set("password", server.getPassword());
                //Kept for compatibility, in case "multiPaths" is not possible or the other end does not support it
                np2.set("path", "/");
                if (paths.size() > 0) {
                    np2.set("multiPaths", paths);
                    np2.set("pathNames", pathNames);
                }
                device.sendPacket(np2);
                return true;
            }
```

这里能看到是把需要的信息都给电脑发送过去了，账号密码是什么呢？

```Java
static final String USER = "kdeconnect";
passwordAuth.password = RandomHelper.randomString(28);
```

呜呼，密码是随机的

那么换个思路，既然电脑端最终是会打开一个`sftp://`协议的`uri`，那就试着注册一个`sftp`的协议？



## 注册sftp协议

类似于`sftp://`这样的协议有一个众所周知的例子，就是`thunder://`迅雷的链接。那么他是怎么实现的呢？

众所周知，注册表就是操作系统的配置中心，那么里面肯定会有`thunder`这样的关键字，通过`win+r`运行`regedit`打开注册表编辑器，然后搜索`thunder`

![image-20210916082815571](https://qiniusave.xint.top/mdimage-20210916082815571.png)

出现很多这样的结果，例如上面这个应该就是右键菜单。继续搜索，直到看到了一个可疑的项。搜索过程中，逐渐发现`HKEY_CLASSES_ROOT`这个项里面都是指定什么协议或者什么文件由什么程序处理类似的的信息。

最终找到了下面的信息，一目了然

![image-20210916083225240](https://qiniusave.xint.top/mdimage-20210916083225240.png)

简单解释就是，浏览器的地址栏输入`thunder://xxxxx`这样的链接之后，去执行`Thunder.exe`这个程序，并且传递参数，`%1`和`-StartType:thunder`其中前者是个变量类似于`Linux`函数中的`$1`，代表的是`thunder://`后面的字符串。再加上第二个参数，`Thunder.exe`就能知道用户是通过`thunder://`协议想要下载`xxxxxx`链接的文件。

原理知道了，那么接下来只需要按照这样的格式创造一个`sftp://`协议的就好了。那么谁来处理这个链接呢？肯定是`FileZilla`啦，一个很成熟的`ftp`协议的开源实现，所以肯定是支持参数启动的。

`FileZilla`的官方没有找到对应的描述，但是在别的地方看到了启动方式[filezilla 怎么带这参数启动 - SegmentFault 思否](https://segmentfault.com/q/1010000015270795)

`"E:\filezilla.exe" ftp://username:password@ftp.server2.com --local="F:\electron"`

看上面的参数，第一个参数应该就是`uri`了，我猜测电脑端的`kconnect`就是尝试通过`IE`框架运行的这么个东西。第二个参数就是指定本地文件夹了，看起来不指定也可以。

## 注册sftp协议

根据我的经验，最好的办法就是，把`thunder`协议的配置信息导出，然后修改成自己的就行了，否则哪个地方出错了会比较烦人。

```ini
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\thunder]
"URL Protocol"=""

[HKEY_CLASSES_ROOT\thunder\Shell]

[HKEY_CLASSES_ROOT\thunder\Shell\Open]

[HKEY_CLASSES_ROOT\thunder\Shell\Open\command]
@="\"C:\\Program Files (x86)\\Thunder Network\\Thunder\\Program\\Thunder.exe\" \"%1\" -StartType:thunder"

```

这是`thunder`对应的配置

修改定制一下

```ini
Windows Registry Editor Version 5.00

[HKEY_CLASSES_ROOT\sftp]
"URL Protocol"=""

[HKEY_CLASSES_ROOT\sftp\Shell]

[HKEY_CLASSES_ROOT\sftp\Shell\Open]

[HKEY_CLASSES_ROOT\sftp\Shell\Open\command]
@="\"C:\\Program Files\\FileZilla FTP Client\\filezilla.exe\" \"%1\""
```

现在另存为`sftp.reg`，双击导入到注册表。现在再通过`kconect`位于右下角托盘图标右键的`浏览设备`就能直接打开`FileZilla`了，并且能直接连接上手机了。

![image-20210916084902145](https://qiniusave.xint.top/mdimage-20210916084902145.png)

顺利的不可思议。。。感谢伟大的`FileZilla`，开源免费真的是太好了。



## 注意

手机端的`kconnect`需要在插件中设置一个目录作为`sftp`的工作目录，不然电脑端点击浏览设备什么都不会发生
