---
title: "协程学习过程"
date: 2021-01-12T21:07:42+08:00
tags : ["Python","网络","协程","编程"]
categories : ["Python"]
---

# 开始

之前一直在做那个`rProxy`的项目，后来发现，服务端不用协程或者异步编程这样的手段是不行的，最主要的问题就是对于每个http请求都对应一个线程，这个开销非常大。对于一个网页而言，四五十个http请求已经是非常常见的事情了，如果有很多个客户端，一下子线程数可能得有几百个。而Python的多线程众所周知的虚假。

所以我就会考虑使用异步套接字来做。

在异步套接字中会需要一个主循环用来管理套接字，手动管理比较麻烦，后来了解到了协程。发现这个就是非常适合做这个事情的啊。

而Python的库就是`asyncio` 官方文档在这里
	
<[协程与任务 — Python 3.9.1 文档](https://docs.python.org/zh-cn/3/library/asyncio-task.html)>
	
不过后面又发现了一个更神奇的东西，就是`gevent`这个库，项目在这里
	
<[GitHub - gevent/gevent: Coroutine-based concurrency library for Python](https://github.com/gevent/gevent/)>
	
pip不能安装的话`whl包`在这里[Python Extension Packages for Windows - Christoph Gohlke (uci.edu)](https://www.lfd.uci.edu/~gohlke/pythonlibs/)



# 协程

线程的话，已经很熟悉了。对于需要大量IO操作的程序，基本上首选多线程。但是线程也有一些缺点，例如说需要考虑线程安全问题，线程间切换也会损耗性能。
	
非常常见的现象就是，使用Python多线程写爬虫我的电脑大约80线程就已经是极限了，再增加线程也不会有什么速度上的加快了。
	
此时，协程就出现了
	
一个线程可以包含很多的协程，而协程之间的切换不需要经过系统，不需要进行内核态的切换。还有各种复杂的保存现场。



# 学习过程
这里就记录一下对协程的学习过程，主要是两种。
	
一个是以`asyncio`为主的协程
	
另一个是以第三方库`gevent`为主的协程
	
## asyncio

`asyncio`是最先了解到的技术，可以用于实现协程。一个简单的Demo如下。

```Python
import asyncio as ayc

i = 0

async def p(c):
    global i
    while True:
        print(f"t{c}: {i}")
        i+=1
        await ayc.sleep(0)
    
async def fun():
    t1 = ayc.create_task(p(1))
    t2 = ayc.create_task(p(2))

    await t1
    await t2

def main():
    ayc.run(fun())

if __name__ == "__main__":
    main()
    
```

这个例子只是对一个公共变量进行叠加，结果
	
![](https://raw.githubusercontent.com/LonelySinging/img/master/1236187-20210112224311186-2091985428.png)

运行之后能够看到数字递增，而且非常有序，t1,t2两个协程分工交替出现
	
而多线程不加锁的情况下，很容易出现以下的情况。也就是出现t1 t2时间分配不均匀的情况，并且线程不安全 `89914`出现在了`89916`后面

![](https://raw.githubusercontent.com/LonelySinging/img/master/1236187-20210112224337464-696360677.png)
多线程代码如下

```Python
import threading

i = 0

def fun(c):
    global i
    while True:
        print(f"t{c}: {i}")
        i+=1
    
def main():
    threading.Thread(target=fun,args=(1,)).start()
    threading.Thread(target=fun,args=(2,)).start()
    
if __name__ == "__main__":
    main()
```



目前到这里就没有问题，但是等我实际写代码的时候，发现一个问题。
	
也就是，如果一个IO操作阻塞，则整个线程卡死，导致协程不能正常切换。例如下面的代码

```Python
import requests
import asyncio
import time

urls = ["https://www.baidu.com","https://www.cnblogs.com/","https://blog.csdn.net/"]

async def getHtml(url):
    stime = time.time()
    r = requests.get(url)
    print(f"{url}, {r.status_code}, {time.time() - stime}")
    
async def main():
    tasks = []
    for url in urls:
        tasks.append(asyncio.create_task(getHtml(url)))
    
    stime = time.time()
    await asyncio.wait(tasks)
    print(f"Done, {time.time() - stime}")
    
if __name__ == "__main__":
    asyncio.run(main())
```

代码作用是分别请求三个域名，记录响应时间以及总时间
	
结果如下，注意，此时虽然用了协程，但是总时间却是三次请求时间之和。

![](https://raw.githubusercontent.com/LonelySinging/img/master/1236187-20210112224515840-470165617.png)

原因就是，直接使用`asyncio`实现协程并不会监听IO阻塞情况，也就是在`requests.get()`的时候，协程没有切换。导致整个线程阻塞。
	
所以实际的执行流依旧是串行执行，那么协程就毫无意义。

### aiohttp

而这个问题存在的原因就在于，`requests`这个库是`同步库`，底层是`同步socket`。对应的，另一个名为 `aiohttp`的第三方库是基于`asyncio`开发的http库可以解决这个问题。
	
`aiohttp`官方文档如下
	
<[Welcome to AIOHTTP — aiohttp 3.7.3 documentation](https://docs.aiohttp.org/en/stable/)>
	
一个Demo如下

```Python
import aiohttp
import asyncio
import time

urls = ["https://www.baidu.com","https://www.cnblogs.com/","https://blog.csdn.net/"]

async def getHtml(url):
    print(f"请求: {url}")
    stime = time.time()
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            print(f"status: {response.status}, {time.time() - stime}")
            # html = await response.text()
            # print(f"Body: {html[:20]}")
          
async def main():
    tasks = []
    for url in urls:
        tasks.append(asyncio.create_task(getHtml(url)))
    await asyncio.wait(tasks)
    print("Done")

asyncio.run(main())
```

最后结果如下

![](https://raw.githubusercontent.com/LonelySinging/img/master/1236187-20210112224537300-171438382.png)


可以看到总花费时间近似于最长请求时间。也就是完成了异步请求

那么，到这里似乎就已经圆满了。但是，我还了解到了`gevent`这个库



## gevent

直接贴代码

```Python
from gevent import monkey; monkey.patch_all()
import gevent
import requests
import asyncio
import time

urls = ["https://www.baidu.com","https://www.cnblogs.com/","https://blog.csdn.net/"]

def getHtml(url):
    stime = time.time()
    r = requests.get(url)
    print(f"{url}, {r.status_code}, {time.time() - stime}")
    
def main():
    tasks = []
    for url in urls:
        tasks.append(gevent.spawn(getHtml,url))
    
    stime = time.time()
    gevent.joinall(tasks)
    print(f"Done, {time.time() - stime}")
    
if __name__ == "__main__":
    main()
```

结果如下
![](https://raw.githubusercontent.com/LonelySinging/img/master/1236187-20210112224554894-620598290.png)


看到上面的结果和使用`aiohttp`库一样，并且依旧是用了`requests`这个库，而前面说过，requests是同步库。。。 是不是很神奇！是的，给我惊讶到了。
	
然后原理也给了解了一下， <[关于 python gevent 架框 作为 TCP服务器 的 代码问题 ， 每个 socket 的 消息 接收 是否有使用 事件监听回调的方法呢？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/20703476/answer/15911452)>
	
总的来说，就是替换了Python自己的socket实现，把socket设置成了异步。绝了~