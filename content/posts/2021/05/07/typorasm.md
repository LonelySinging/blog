---
title: "typora文档复制到博客图片失效 SM.MS限制"
date: 2021-05-07T05:57:10+08:00
tags : ["Python","Typora","图床","折腾"]
categories : ["博客修建"]
---

## 开始

今天开始尝试使用 `Typora` 写markdown 然后复制到博客园，不过会有一个问题

那就是 typroa 插入的图片都是本地的，md文档复制到博客园之后，图片都失效了

通过百度，有工具可以直接把 md 文档中的图片上传到博客园，然后替换文档中的链接，我把工具下载下来之后，发现这东西依赖`.net` ，而这个东西好大，所以萌生了自己写一个工具的想法



## 注意!!!!!

sm.ms图床有限制

+ 一分钟10张
+ 一小时20张
+ 一星期50张

所以,,, 白嫖不了了。。。

其实是可以的 只是需要改造一下... 

加一个代理池

但是  还是默默的 换成自己的把



## 分析博客园上传图片接口

> 博客园有三种编辑器，其中`markdown` 编辑器和`tinyMCE` 编辑器可以上传图片

1. 首先分析一下接口，在拖入图片之后会请求一个接口，上传文件，但是我用python 仿照他请求之后一直返回500错误。另一个编辑器使用的接口也是不可用的。
2. 最后折腾了三四个小时发现它在服务端设置了禁止跨域.... 我早该想到的 !
3. 所以不能使用这种方式了，但是已有的.net 工具是怎么实现的呢？应该是官方发布的吧。不然登录验证怎么做



## 寻找图床

既然博客园是不能上传图片了，那就需要找一个图床，需求就是特别稳定，据说新浪图床还炸了。找了之后有三个图床。

+ sm.ms
+ 路过图床
+ 中关村的图片上传接口
+ 找到的其他接口都需要登录才行，所以不好使

1. 简单分析之后，决定使用sm.ms ,因为这个网址没有对接口做演示，且官方也放出了api,并且也没有什么限制
2. 上传文件之后它请求了这个接口

![image-20200228114035021](https://qiniusave.xint.top/img/dPvhbAXFLuCkDjH.png)

3. 并且返回了一个json 数据是图片的url (这么老实的图床，感觉白嫖的良心痛，但是也不能一直白嫖吧)
4. 写了python 代码测试了一下接口，确实可用

```python
import requests as rq
import json

# 传入图片名 返回图床url
def uploadImg(imgname):
    url = 'https://sm.ms/api/v2/upload?inajax=1'
    headers = {
    'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.122 Safari/537.36 Edg/80.0.361.62',
    }

    files = [('smfile',(imgname,open(imgname, 'rb'),'image/png'))]
    r = rq.post(url,headers=headers,files=files)
    dic = json.loads(r.text)
    print(r.text)
    try:
        if 'data' in dic:
            return dic['data']['url']
        elif 'code' in dic:
            if dic['code'] == 'image_repeated':
                return dic['images']
        else:
            print(r.text)
            exit()
    except Exception as e:
        print(str(e))
        exit()

print(uploadImg('2.png'))

```

5. 那后面就是，正则匹配图片链接，然后，通过上面的自定义函数得到图片在图床的url,再替换到md文件中即可
6. 最后效果

![image-20200228114705882](https://qiniusave.xint.top/img/l8zqLFwbVBgu1AK.png)

7. 此时文件中的链接被替换成了图床链接，这时候文档复制到哪里都行了，不过为了防止图床炸掉，还是在本地保存了一份 (将typroa 图片设置为复制到文档同目录就好了)
8. 最后贴出全部代码

```Python
import requests as rq
from bs4 import BeautifulSoup
import sys
import re
import json

def uploadImg(imgname):
    url = 'https://sm.ms/api/v2/upload?inajax=1'
    headers = {
    'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.122 Safari/537.36 Edg/80.0.361.62',
    }

    files = [('smfile',(imgname,open(imgname, 'rb'),'image/png'))]
    r = rq.post(url,headers=headers,files=files)
    dic = json.loads(r.text)
    try:
        if 'data' in dic:
            return dic['data']['url']
        elif 'code' in dic:
            if dic['code'] == 'image_repeated':
                return dic['images']
        else:
            print(r.text)
            exit()
    except Exception as e:
        print(str(e))
        exit()

def getImgStr(str_):
    pattern = re.compile(r'\!\[.*?]\(.*?\)')
    return pattern.findall(str_)

def getImgName(img_str):
    pattern = re.compile(r'\(.*.\)')
    return pattern.findall(img_str)[0][1:-1]

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print('请传递参数进来'+str(len(sys.argv)))
        exit()
    md_filename = sys.argv[1]
    md_str = ''
    with open(md_filename,'r',encoding='utf-8') as f:
        md_str = str(f.read())

    img_strs = getImgStr(md_str)
    for img_str in img_strs:
        img_name = getImgName(img_str)
        img_url = uploadImg(img_name)
        md_str = md_str.replace(img_name,img_url)
        print(img_url)

    with open('修正_'+md_filename,'w',encoding='utf-8') as f:
        f.write(md_str)

```

9. 最后再通过pyingtaller 打包成exe 然后放到系统变量path所指向的目录下就行了。下次编辑文档，在md文档所在处，地址栏输入cmd 然后执行命令 `up_img 文档名.md` 就可以完成图片上传替换



## 额外的

虽然图片不能上传到博客园，但是解析接口的时候发现了些比较诡异的事情。
	
`TinyMCE`编辑器所使用的图片接口，上传之后没有返回任何东西，但是图片却加载进来了，我死活不相信这么玄学的事情，分析了js代码之后，确实看到了对接收结果的处理，但是浏览器调试工具就是不显示返回结果。
	
最后使用fiddler 抓包，终于发现了数据

![image-20200228115610372](https://qiniusave.xint.top/img/2x8zWlfCcyQwi7E.png)

显然是加密的，但是如何做到 `dev-tool`不显示，还真的不知道的骚操作
	
然图床可用，但是终究放心不下，最终准备使用七牛云，毕竟免费送10G空间流量的不是...