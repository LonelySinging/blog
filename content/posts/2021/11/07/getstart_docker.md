---
title: "Docker的使用记录"
date: 2021-11-07T17:17:15+08:00
draft: false
tags: ["docker","折腾","工具"]
categories: ["折腾"]
---

# 开始
这是第一个尝试在`Leanote`上面编写文章，我觉得最重要的事情就是能够保证`md`文件是能够移植的，否则如果这个软件不靠谱的话，我还能把文章移动到别的地方去。所以先写一篇文章看看效果如何，方便不方便移动。

----------
一直听说`docker`很好用，但是并没有把他作为一个主力使用的工具，因为看它的命令还是很繁琐的，，，说起来，，确实是因为我没有耐心的去看它的使用方式。那么接下来就通过文字的方式简单的记忆一下`docker`的使用方式，以及简单的工作原理。

--------
# 基本原理
实际上`docker`就是容器，类似于`proot`的存在。对于`proot`的文件系统来说，就是`docker`的镜像(`image`)。这个镜像可以通过`docker pull`的方式从镜像源拉取，类似于`apt`或者`yum`的工作方式，非常方便，当然还是需要换源的。想要知道本地有什么已经下载的镜像可以通过`docker images`命令查看。
把镜像运行起来之后，就称之为容器。当前有什么正在运行的容器可以通过`docker ps`命令查看。
容器之间都是相互独立的存在，彼此是不能相见的，这也对安全了些保障，如果是正常情况下直接在主机上面运行一个网站，那么可能会出现文件上传漏洞这样的问题，进而被控制整个主机。但是如果是通过`docker`来运行网站，那么就不需要担心这个问题了，数据库和网站都在容器里面，别的什么东西都没有，也是完全不用担心的。
如果想要让两个容器互相通讯，使用`--link`参数就可以了，方法是`https://www.cnblogs.com/vincenshen/p/8724584.html`

# 命令记录
## 安装
建议直接通过`apt`安装，不添加仓库，因为会导致每次`apt upgrade`都得去官方仓库会非常慢。
通过`sudo apt install docker`和`sudo apt install docker.io`安装就行了

## 换源
编辑`/etc/docker/daemon.json`
替换成如下

```ini
{"registry-mirrors" : [
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com",
    "https://cr.console.aliyun.com/"
    ]}
```
然后重启服务就好了 `service docker restart` 或者新版`ubuntu`使用`systemctl restart docker`

## 运行
通过`docker run`命令就可以运行了，例如使用下面的命令
`docker run -d --name my_nextcloud -v /data/nextcloud:/data/data -p 8080:80 -e ENV1=11 nextcloud`
上面的命令含义是
运行 `nextcloud`容器，如果本地没有这个镜像就去仓库拉取

+ `-d`参数表示后台运行，不会在控制台有输出
+ `--name my_nextcloud`表示给这个容器取了一个别称
+ `-v`表示与宿主机共享目录`:`前面是宿主机，后者是容器
+ `-p` 表示端口映射，`:`前面是宿主机，后者是容器，运行后访问宿主的`8080`相当于访问容器的`80`端口
+ `-e` 设置环境变量，多个变量多个`-e`这个选项一定要在容器名之后

更多参数看`https://www.cnblogs.com/yfalcon/p/9044246.html`
如果使用`-d`启动失败了，可以通过`docker logs 容器id`看到输出

## 查看镜像或容器
查看所有镜像`docker images`
查看运行中的容器`docker ps`
查看所有容器`docker ps -a`

## 删除镜像或容器
### 删除镜像
删除镜像需要保证这个镜像没有对应容器在运行，删除时如果说不能删除，可以通过添加`-f`参数强制删除例如
`docker rmi -f 容器id或容器名`
### 删除容器
`docker rm 容器id或容器名`
删除所有退出的容器
`docker rm $(docker ps -a | grep Exited | awk '{print $1}')`

## 停止容器
`docker stop 容器id或者容器名`
需要注意每次运行容器的时候`id`是会变化的，但是镜像`id`是不变的

## 启动停止的容器
`docker start 容器`

# 连接两个容器例子
例如此时需要使用一个数据库容器，还有一个`ubuntu`容器，需要在`ubuntu`中连接数据库容器中的数据库

+ `docker run -d --name="my" -e MYSQL_ROOT_PASSWORD="123456" mysql` 
    + 启动一个名称为`my`的`mysql`容器，指定密码是`123456` 且是后台运行
+ `docker ps`能够看到`mysql`容器运行起来了，如果没有通过`docker logs my`看到输出，就知道错误原因了
+ `docker run -it --link my ubuntu` 启动一个`ubuntu`容器，并且通过`--link`参数连接`my`，其中`-it`是两个参数，但综合使用效果就是进入容器并且获得里面的`shell`控制台
+ 进入`ubuntu`控制台之后
    + 通过`apt update` 更新软件列表否则不能`apt`东西
    + `apt install mysql-client`安装`mysql客户端`
    + `mysql -h my -uroot -p` 然后输入密码`123456`应该就能连接上了

此时通过`cat /etc/hosts`就能看到 `172.17.0.2      my 18f749131a95` 这样的东西，可见上面通过`-h`参数指定的主机`my`对应的就是另一个容器的`ip`地址
    
# 注意
+ 如果不通过`--name`参数指定容器名则删除或者启动操作不能使用容器，注意区分容器名和仓库名
+ 如果没有必要，那么操作的时候把容器名或者镜像名放到命令的最后能避免很多错误 (`docker`的参数对顺序敏感)
