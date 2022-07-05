+++
title = "RAT病毒分析"
date = 2019-08-02T20:37:11+08:00
tags = ["病毒","逆向分析"]
categories = ["逆向分析"]
+++

[样本来源](https://bbs.kafan.cn/thread-2178386-1-1.html)


# 概述

> 好久没有分析练习了



# PEID

![image-20200409170429252](https://qiniusave.xint.top/img/image-20200409170429252.png)

> 应该是C/C++开发的，节表四个很正常

![image-20200409170541686](https://qiniusave.xint.top/img/image-20200409170541686.png)

> 透明图标，大小136KB 139,264 字节



# 火绒剑

![image-20200409171108334](https://qiniusave.xint.top/img/image-20200409171108334.png)

>  没有其他多余的操作,打开之后直接开始尝试连接主机
>
> 通过ping，确认服务器是活着的，但是服务端有没有不知道
>
> 119.167.182.164:15950



# 服务器探索

## nmap

> 通过命令 `nmap -T4 -O -A -v 119.167.182.164` 检查服务器开启端口

### 端口

```shell
Discovered open port 8080/tcp on 119.167.182.164
Discovered open port 3389/tcp on 119.167.182.164
Discovered open port 80/tcp on 119.167.182.164
Discovered open port 9/tcp on 119.167.182.164
Discovered open port 2002/tcp on 119.167.182.164
Discovered open port 1433/tcp on 119.167.182.164
Discovered open port 2107/tcp on 119.167.182.164
Discovered open port 2009/tcp on 119.167.182.164
```

> 看到开启了 8080 但是访问无效
>
> 3389 可能是windows服务器
>
> 80 访问也是无效
>
> 1433 表示安装了mssql?
>
> 1433/tcp open  ms-sql-s   Microsoft SQL Server  14.00.1000.00
>
> | ms-sql-ntlm-info: 
>
> |   Target_Name: WIN-EIK3KA7QGQO
>
> |   NetBIOS_Domain_Name: WIN-EIK3KA7QGQO
>
> |   NetBIOS_Computer_Name: WIN-EIK3KA7QGQO
>
> |   DNS_Domain_Name: WIN-EIK3KA7QGQO
>
> |   DNS_Computer_Name: WIN-EIK3KA7QGQO
>
> |_  Product_Version: 10.0.14393

### 操作系统

> Aggressive OS guesses: Linux 2.6.18 - 2.6.22 (91%), Microsoft Windows Server 2016 (90%), Microsoft Windows Vista Home Premium SP1 (90%), Microsoft Windows Server 2008 R2 (89%), Microsoft Windows 10 1511 - 1607 (87%), Microsoft Windows 10 1607 (87%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (87%), Microsoft Windows Server 2008 (86%), Microsoft Windows 7 SP1 (86%), Microsoft Windows 7 or Windows Server 2008 R2 (86%)

nmap认为91%可能是linux ,但是我感觉还是windows server，一般来说 开了3389 和 安装了 mssql 的话，windows 可能性更大

## wrieShark

![image-20200409172145543](https://qiniusave.xint.top/img/image-20200409172145543.png)

> 可以看到，程序尝试连接服务端 但是 服务端返回了 RST 标志，那么服务端可能没有运行



# IDA静态分析

> 上了就遇到一个下马威，遇到不少问题
>
> + C++编写的代码，异常处理一大堆
> + sp-analysis failed...  解决方式<https://www.cnblogs.com/cjdty/p/12671705.html>



