---
title: "ffmpeg 使用记录"
date: 2021-10-11T08:41:32+08:00
draft: false
tags: ["ffmpeg","视频相关","windows"]
categories: ["折腾"]
---

这周周末尝试把我硬盘上面的视频文件压缩了一下，但是效果并不理想。其中主要有两个原因，

+ 视频本来就是`h264`的编码，再重新编码也没啥用，因为限制大小的主要是码率
+ `ffmpeg GPU`加速版的`h265`编码器有些问题

导致有些文件使用`hevc_nvenc`编码器会编码失败，最终不能播放。所以最终还是重新压制了`h264`，但是这么干最终只是把码率降了下来。最终使用命令如下

`ffmpeg -i 1.mp4 -c:v xlib264_nvenc -y 2.mp4`

经过各种查询资料之后发现默认的参数就差不多是最适合我的了，如果要指定码率，那就需要考虑每个原文件的码率都不一样，这样的话压缩之后的目标码率也不一样。而`ffmpeg`的默认参数把码率都放到了`2300`左右，能看到有损失画质，但是大概可以接受吧 (手机播放的话)

> 首先说明一个困惑我的问题
>
> `AVC` 就是`h264`
>
> `HEVC` 就是`h265`

## 一些比较重要的参数举例

首先举个例子，这个例子里面就包含了几个比较常用的参数

`ffmpeg -i a.mp4 -c:v libx264 -preset veryslow -crf 18 -c:a copy b.mp4`

### `ffmpeg`

这个也算哈，去这里下载[Download FFmpeg](http://www.ffmpeg.org/download.html)，然后为了方便起见我会把这个文件放到我的用户目录，也就是`Users\Administrator\`然后添加系统变量，让我哪里都能用。

### `-i`

它后面的`a.mp4`就是输入的文件，最后面`b.mp4`是输出的文件

### `-c:v`

这个是重点，后面的参数表示使用什么编码器，具体有什么编码器使用`ffmpeg -codecs` 看所有编码，比如说`h264`的条目

```Shell
DEV.LS h264                 H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10 (decoders: h264 h264_qsv h264_cuvid ) (encoders: libx264 libx264rgb h264_amf h264_nvenc h264_qsv nvenc nvenc_h264 )
```

+ 最前面表示的是属性，有以下几个属性

	```Shell
	 D..... = Decoding supported	// 支持解码
	 .E.... = Encoding supported	// 支持编码
	 ..V... = Video codec			// 视频编码
	 ..A... = Audio codec			// 音频编码
	 ..S... = Subtitle codec		// 字幕编解码器 (没用过不清楚)
	 ...I.. = Intra frame-only codec// 仅帧内编解码器 (没用过不清楚)
	 ....L. = Lossy compression		// 有损压缩
	 .....S = Lossless compression	// 无损压缩
	```

+ `h264`是编码名称，后面`H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10` 应该都是别称

+ `decoders` 是解码器 (`h264 h264_qsv h264_cuvid`)

+ `encoders` 是编码器 (`libx264 libx264rgb h264_amf h264_nvenc h264_qsv nvenc nvenc_h264`)

后面这个编码器是比较重要的属性，就是可以跟在`-c:v`参数后面的值，比如我想压缩成`h264`编码，可以写`libx264`，如果是`h264_nvenc` 表示使用`GPU`加速，需要注意的是英伟达的显卡是`h264_nvenc`如果是`AMD`的显卡应该是`h264_amf` 其他的几个选项不清楚，没有使用过。

通过上面的内容就知道了编码器有什么了，接下来优化参数怎么写呢？

### `-preset`

这个是编码器的参数，一般来说有以下取值

| preset值  | 释义   |
| --------- | ------ |
| ultrafast | 极快   |
| superfast | 超快   |
| veryfast  | 非常快 |
| faster    | 更快   |
| fast      | 快     |
| medium    | 中     |
| slow      | 慢     |
| slower    | 更慢   |
| veryslow  | 非常慢 |
| placebo   | 超慢   |

**但是好像不同的版本支持是不一样的，所以，需要自行查看取值，通过`ffmpeg -h encoder=libx264`	查看编码器的帮助**

如果选择特别慢，那么编码就会很慢，但是文件会相对小一点 (是这么说，但是我似乎把一个`2GB`的文件编码成了`36GB`码率`124M`，还不是很清楚为啥)



比如我执行完毕之后可以知道下面的信息

```Shell
  -preset            <string>     E..V....... Set the encoding preset (cf. x264 --fullhelp) (default "medium")
  -tune              <string>     E..V....... Tune the encoding params (cf. x264 --fullhelp)
  -profile           <string>     E..V....... Set profile restrictions (cf. x264 --fullhelp)
  -fastfirstpass     <boolean>    E..V....... Use fast settings when encoding first pass (default true)
  -level             <string>     E..V....... Specify level (as defined by Annex A)
  -passlogfile       <string>     E..V....... Filename for 2 pass stats
  -wpredp            <string>     E..V....... Weighted prediction for P-frames
  -a53cc             <boolean>    E..V....... Use A53 Closed Captions (if available) (default true)
  -x264opts          <string>     E..V....... x264 options
  -crf               <float>      E..V....... Select the quality for constant quality mode (from -1 to FLT_MAX) (default -1)
  -crf_max           <float>      E..V....... In CRF mode, prevents VBV from lowering quality beyond this point. (from -1 to FLT_MAX) (default -1)
  -qp                <int>        E..V....... Constant quantization parameter rate control method (from -1 to INT_MAX) (default -1)
```

总之是相当多，每个编码器不同，支持的选项都不一样，最好通过`ffmpeg -h encoder=libx264` 查看一下，否则等了很久转码完毕之后发现参数不对就很浪费时间。

### `-crf`

是用来控制码率的，一般取值是在`0-51`，据说`18-28`是比较常见的取值范围，数字越小质量越好。

我编码的时候`-crf 25`码率好像在`2000k`左右，具体不知道它怎么确定码率的。

## 使用`GPU`加速

需要说明的是`GPU`加速是编码器的功能，通过`ffmpeg -codecs`列出所有的编码器，然后找有`nvenc`字样的编码器，比如`h264_nvenc` 这个编码器就是使用了`GPU`的`h264`编码器，要注意这个编码器和`libx264`的优化参数不一样，需要去专门看它的优化参数。

> 可以通过 `ffmpeg -codecs | grep nvenc`找出来，`windows`使用`ffmpeg -codecs | findstr nvenc` 命令 (我看好像就`h265`和`h264`有对应的`GPU`加速版本 分别是`hevc_nvenc`和`h264_nvenc`)



## `python`获取视频码率

首先需要把`ffprobe.exe`的路径添加到系统变量里面，基本思想是通过`ffprobe`命令得到视频信息，通过参数指定输出结果是`json`格式，通过管道拿到结果，然后`json.loads()`得到字典。

```Python
import os,json
import shlex
from subprocess import Popen, PIPE

def getJsonString(strFileName):
    strCmd =  'ffprobe.exe -v quiet -print_format json -show_format -show_streams -i "' +  strFileName  + '"'  
    # print (strFileName)
    cmd = shlex.split(strCmd)
    p = Popen(cmd, stdout=PIPE, stderr=PIPE)
    o, e = p.communicate()
    # print(o.decode("utf-8"))
    
    mystring = o.decode("utf-8")
    # mystring = os.popen(strCmd).read().encode('gbk')
    # mystring = os.popen(strCmd).read().decode('gbk')
    # mystring = os.popen(strCmd).buffer.read().decode(encoding='utf-8')
    
    return  mystring
    
def get_rate(file_name):
    info = getJsonString(file_name)
    info = json.loads(info)
    # print (info["format"]["bit_rate"])
    return int(info["format"]["bit_rate"])

print(get_rate(r"C:\Users\Administrator\Documents\1\视频\001.mp4"))
```

上面代码是能运行的，其中我使用了`os.popen()`但是如果命令执行结果返回了中文，就会出现编码错误。所以根据网上建议使用了`subprocess`里面的管道。



---

总之，`ffmpeg`的使用是一个相当复杂的事情，需要花很久去研究参数，上面只是一些简单的参数，执行了`ffmpeg -h encoder=libx264`这样的帮助命令之后就知道其内容有多恐怖了，而这个命令显示的还只是一个编码器的参数。。。

---

**相关参考**

[[win10\] ffmpeg gpu加速_THE XING-CSDN博客_ffmpeg gpu加速](https://blog.csdn.net/qq_39575835/article/details/83826073)

[ffmpeg参数中文详细解释_雷霄骅(leixiaohua1020)的专栏-CSDN博客_ffmpeg参数详解](https://blog.csdn.net/leixiaohua1020/article/details/12751349)

[ffmpeg 码率控制（总结篇）_隔壁老王呀的专栏-CSDN博客_ffmpeg 码率](https://blog.csdn.net/Martin_chen2/article/details/105772872)

*最好的帮助是帮助命令*
