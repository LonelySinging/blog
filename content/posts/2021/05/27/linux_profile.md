---
title: "linux的登录脚本记录"
date: 2021-05-27T17:52:22+08:00
draft: false
tags: ["杂项","linux","备忘"]
categories: ["备忘"]
---




> 通过在`Linux`的启动脚本中添加一些指令，可以实现一些简单的功能，此处记录一下 未来换设备的话能直接使用



## 打开core文件生成开关

```Shell
ulimit -c unlimited
```

这样在程序崩溃之后能够得到`core`文件，以便通过`gdb`调试



## 回收站的实现

在`windows`上面直接删除文件会把文件放到回收站中，以此对手残的用户进行保护

但是`linux`上面就没有，大概`linux`的设计哲学是认为用户足够聪明吧(毕竟能使用`linux`的大概都不是什么电脑小白) 但是每次`rm`掉重要东西之后，还是会心脏骤停。

所以，使用`mv`代替`rm`往往是个好主意

首先先找个位置创建回收站的文件夹，我会直接选在`home/.Recycle` 

使用命令 `mkdir .Recycle` 创建一个文件夹出来

### 删除

```Shell
alias rrm="mv -f -t ~/.Recycle"
```

不直接重载`rm`的原因是因为有时候别人的脚本中出现了`rm` 并且带了参数，但是`rm`和`mv`的参数并不是一模一样的，这时候就可能出错。所以使用`rrm`作为删除



### 查看回收站文件

```Shell
alias rrl="ls -la ~/.Recycle"
```



### 清理回收站

```Shell
alias rrc="rm -rf ~/.Recycle/* ~/.Recycle/.*"
```

因为通过`rm -rf ~/.Recycle` 并不能删除掉`.`开头的文件，所以需要使用后者专门用来删除，但是会出现找不到`. ..`的问题，通过重定向错误输出应该可以隐藏这个提示。但是也有可能会忽略别的有用的信息。

### 统计回收站占用空间

```shell
alias rri="du -sh ~/.Recycle"
```

如此以来，就能实现一个简单的回收站，但凡起作用一次，都是一件值得庆幸的事情。

尽管如此，在使用`rm` 之前，还是思考一下比较好。



## ls命令重载

```shell
alias ls="ls --color"
```

这个一般安装好系统就会有，如果没有`--color`参数，文件夹就没有颜色



```shell
alias ll="ls -lh"
```

以更科学的方式显示文件大小，并且查看详情信息，还是非常好用的命令，同样的，一些发行版默认就有这个



```shell
alias la="ls -lha"
```

就是比上一条多了显示隐藏文件的功能



## 超级目录切换

最近修改一个项目，是`java`的，代码文件足足在六层文件夹包裹之下，每次打开一个新的终端，都会很麻烦，所以，想了一个办法。

### 保存路径

```shell
alias cdc="pwd > ~/.cdc"
```

执行之后，会在`home`目录下创建一个隐藏文件，保存当前的工作路径 

### 读取路径

```shell
alias cdd="cd \`cat ~/.cdc\`"
```

这个困扰了我好一会儿，最开始的版本是 

```shell
alias cdd="cd `cat ~/.cdc`"
```

但是发现不管`cdc`保存了什么命令，`cdd`都只能切换到一个固定的目录，直到无意间执行了`alias`之后，发现建立的别名是这样的 

```shell
alias cdd="cd /home/st/"
```

也就是说，在执行`alias`的时候，就已经把``里面的东西解释执行了



## 仅查看自己的进程

公司的服务器上面，并没有`root`权限，通过`ps aux` 看到很多进程，但是都不能操作，反而会影响找到有用的进程，所以会考虑只显示自己的进程

```shell
alias psu="ps aux | grep \"^`id -u` \""
```

这样，之后执行`psu`就只会显示自己的进程了

+ `ps aux`会显示所有的进程，交给`grep`过滤
+ `id -u`将会获取到自己的用户`uid` 
+ `^`表示匹配自己`uid`开头的条目
+ 最后还有一个空格不能少，否则会把包含你`uid`的别人的进程也显示出来

## 命令提示符修改

```Shell
export PS1='\[\e[31;1m\][\u@\[\e[32;1m\]\h \[\e[32;1m\]\W]\$ \[\e[0m\]'
```

将会得到

![image-20210629125123739](https://qiniusave.xint.top/mdimage-20210629125123739.png)

## 快捷搜索

```shell
alias f="find . -name"
```

使用的时候直接使用`f main.cpp`就能快速找到需要的文件了

## 文本搜索

对于`grep`来说，需要传递多个参数，这个时候就不能通过`alias`了，这时候可以选择使用`function` 也是添加到`.bashrc`文件中

```shell
function gp(){
	if [ $# -eq 0 ];then
		echo "Use: gp <string>"
	else
		grep -Irn "$1" *
	fi
}

function gpv(){
	if [ $# -eq 0 ];then
		echo "Use: gpv <string>"
	else
		grep -v "$1"
	fi
}
```

一般的使用方式是`gp main(` 就能在当前目录和子目录中找到相应的文件，如果配合`gpv`则这么用 `gp "main" | gpv "tags"` ，就能过滤掉带有`tags`的行。还是比较好用的。

## 记忆一个目录

经常会一次打开很多个终端，会有保持两个终端工作路径相同的需求。此时可以这么写

```shell
alias cdc="pwd > ~/.cdc"
alias cdd="cd \`cat ~/.cdc\`"
```

假如第一个终端，`cd /etc/vim` 进入了一个目录，然后执行`cdc`

第二个终端执行`cdd`就能直接切换到和第一个终端一样的工作路径 当然，也能写个脚本实现记忆更多，比如记忆常用的工作目录，这个功能有时间再完成吧



## VIM的设置

```shell
syntax on
colorscheme molokai
set nu
set tabstop=4
set softtabstop=4
set shiftwidth=4
set expandtab
set autoindent

set fileencoding=utf-8
set fileencodings=ucs-bom,cp936,GBK,GB2312,windows-1252
set termencoding=utf-8
```

这个设置是我在公司用的上古`vim 7.0`的配置，一般来说，最新版本的 `vim`完全不需要考虑这些。。。

也就需要打开行号显示，还有`tabstop`默认是8比较不习惯需要设置

