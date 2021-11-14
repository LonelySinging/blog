---
title: "工具集备忘"
date: 2021-05-23T20:55:26+08:00
draft: false
tags: ["杂项","工具","备忘"]
categories: ["备忘"]
---

> 有很多有用的工具，经常需要的时候忘记名字是什么了。所以特意写一个清单。



# 软件

## scrcpy

使用 `scrcpy` 可以实现电脑手机同屏，MIUI+出来之前的备选项。当然，还有`虫洞`这个软件也可以实现手机电脑同屏，但是感觉这个开源项目更好一点。

<[Genymobile/scrcpy: Display and control your Android device (github.com)](https://github.com/Genymobile/scrcpy)>

## Netch

这个工具是一个代理工具，可以和小飞机配合，实现代理游戏的功能，感觉和sstap很像，但是据说也不太一样。总之，如果需要代理工具的时候，就可以考虑它了。

<[NetchX/Netch: A simple proxy client like SSTap (github.com)](https://github.com/NetchX/Netch)>



## SSTAP

虽然已经有了`Netch`这个工具，但是还是把这个列出来吧

软件模拟了一个网卡，能够实现非常底层的流量代理

[gzbenx/SSTap-beta-setup: 添加规则请看 (github.com)](https://github.com/gzbenx/SSTap-beta-setup)



## PotPlayer

虽然我知道这个播放器的名字，但是这个清单作为一个汇总目录也是很好的。

[PotPlayer官网最新下载 中文,绿色版-PotPlayer播放器](https://potplayer.org/)



## Screen to Gif

非常棒的gif录屏工具，能力超乎想象。如果没有录取声音的需求，这个非常适合用来录制教程。提供的几种压缩方式压缩效率都很高。

[NickeManarin/ScreenToGif: 🎬 ScreenToGif allows you to record a selected area of your screen, edit and save it as a gif or video. (github.com)](https://github.com/NickeManarin/ScreenToGif)



## NetAssist

windows上面用来调试网络的不二之选

[软件下载-野人家园-物联网技术专家平台 (cmsoft.cn)](http://cmsoft.cn/software.html)



## PicGo

配置typora和七牛云能够很方便的 把md博客里面的图片上传上去

[Typora — a markdown editor, markdown reader. (github.com)](https://github.com/Molunerfinn/PicGo)

## typora

据说是最好用的md编辑工具，但是实际上我觉得它默认主题不好看。。。

[Typora — a markdown editor, markdown reader.](https://www.typora.io/)



带上一个很好用的皮肤

[hliu202/typora-purple-theme at v1.5.3 (github.com)](https://github.com/hliu202/typora-purple-theme/tree/v1.5.3)



## SecureCRT 9.0

是个不错的ssh客户端，功能很多。主要是公司使用这个，，，发现确实不错。

之前还觉得这个很老气来着，，， (相比于比较新的软件，确实老气)

而且好像没有中文唉。

[SecureCRT9.0破解版下载 SecureCRT and SecureFX 9.0.0 官方最新完整版(附破解补丁+安装教程) 32/64位 下载-脚本之家 (jb51.net)](https://www.jb51.net/softs/765110.html)

## XShell

也是个很棒的ssh客户端，但是为什么要用CRT呢？

当然是因为CRT有注册机。。。。

虽然XShell有免费版，但是有不少限制，，，很麻烦

[XSHELL - NetSarang Website](https://www.netsarang.com/zh/xshell/)



## winget

众所周知，在linux上面通过包管理工具，能够非常方便的安装软件。

那么，`winget`就是windows上面的包管理器，使用非常方便。

可惜没有正式版，只能手动安装。不过说起来，预览版的win10已经预装这个了。

例子:

```shell
D:\St>winget install git
已找到 Git [Git.Git]
此应用程序由其所有者授权给你。
Microsoft 对第三方程序包概不负责，也不向第三方程序包授予任何许可证。
Downloading https://github.com/git-for-windows/git/releases/download/v2.31.1.windows.1/Git-2.31.1-64-bit.exe
  ██████████████████████████████  47.5 MB / 47.5 MB
已成功验证安装程序哈希
正在启动程序包安装...
已成功安装
```

然后，直接就安装好了，唯一的问题就是不能实现卸载...(新版`winget`好像有卸载选项了) 只能自己手动卸载。不过好在也不是什么麻烦的事情。



+ github
  + [Releases · microsoft/winget-cli (github.com)](https://github.com/microsoft/winget-cli/releases)

+ 应用商店 (浏览器打开)
  + ms-windows-store://pdp/?productid=9nblggh4nns1

## leanote

蚂蚁笔记，开源的笔记系统，自带一个博客系统。支持`markdown`编写，`windows、linux、android`都有客户端，当然也有网页支持，**支持自建服务器**

安装方法[Leanote 二进制版详细安装教程 Mac and Linux · leanote/leanote Wiki (github.com)](https://github.com/leanote/leanote/wiki/Leanote-二进制版详细安装教程----Mac-and-Linux)



## nextcloud

支持自建服务器的的个人网盘，但是实际上不只是一个网盘，功能巨多。各个平台都有客户端，支持文件同步。

[nextcloud/server: ☁️ Nextcloud server, a safe home for all your data (github.com)](https://github.com/nextcloud/server)



## httrack

整站下载工具 可以把整个网站搬下来，用来保存各类开发手册很方便

[Download HTTrack Website Copier 3.49-2 - HTTrack Website Copier - Free Software Offline Browser (GNU GPL)](http://www.httrack.com/page/2/)