---
title: "破解一个极难的软件"
date: 2021-06-09T22:01:49+08:00
draft: false
---

dwda



# 哭了

目前的进度是

+ 通过内存dump的方式，破壳了。
+ 但是代码混淆严重
+ 通过jd-gui得到的类名似乎并不准确，以至于通过`frida`下hook会找不到类
+ 网络上找到的穿透加固的下hook方法并不好用
+ 通过别的逆向工具得到的类名和jd-gui的类名并不相同
+ 接下来准备尝试通过别的方法dump内存，然后看看能否得到准确的类名
+ 通过`frida`枚举的类名并不完整



未来想要尝试的

+ 尝试通过别的dump方式得到dex，因为通过目前的方式拿到dex的过程看起来并不正常
+ 使用 脱壳工具FRIDA-DEXDump
  + [脱壳工具FRIDA-DEXDump - 吴先雨 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wuxianyu/p/14787071.html)
+ objection使用
  + [objection使用 - 走看看 (zoukankan.com)](http://t.zoukankan.com/c-x-a-p-13493629.html)
  + 实际上前者是其插件
+ 一篇看起来大杂烩的文章
  + [转载：实用 FRIDA 进阶 --- objection ：内存漫游、hook anywhere、抓包_freeking101的博客-CSDN博客](https://blog.csdn.net/freeking101/article/details/107749541)
+ 