title: python脚本实现图片压缩
date: 2016/9/10 16：00

categories:
- Python
tags:
- python
---

#前言

在我们android开发过程中，无可避免地都需要对图片资源进行压缩，尽可能减少最终apk包的大小。目前压缩的方法可谓是五花八门，笔者这里主要介绍利用python脚本进行图片的批量压缩。<!-- more -->

#Tinypng

打开它的官网 [https://tinypng.com](https://tinypng.com/) 发现其提供线上上传图片进行压缩，看到例子介绍压缩率还是蛮可观的，而且压缩前后对图片影响不大。我们点进去***DEVELOPER API***,可以看到其支持不少编程语言进行压缩：

![](http://i.imgur.com/Kw8u1l6.png)

为了和下面介绍的方法统一，这里介绍python语言的使用，笔者用的是windows系统，下面介绍请参考windows配置。

1.首先，需要申请一个API key配置：

![](http://i.imgur.com/wSSlLG0.png)
2.接着安装tinify的python插件，在cmd命令行输入：

	pip install --upgrade tinify

3.现在就可以开始编写python脚本了，贴上核心代码： 

	import tinify
	....
	tinify_keys = ""YOUR_API_KEY""
	...
	source = tinify.from_file("unoptimized.jpg")
	source.to_file("optimized.jpg")

Tinypng还有很多对图片进行处理的接口，这里主要是介绍压缩就没有展开细说。其实利用Tinypng对图片进行压缩还是蛮简单的，但是蛋疼的是申请的API key每个月只支持500张的免费压缩：

![](http://i.imgur.com/SQeVvT0.png)

这对于我们开发人员来说是远远不够的（当然，你可以申请多个API key或者直接付钱），所以就得继续寻找看有没有更好的替代方案。

#pngquant

pngquant是国外一个有损的PNG压缩库，介绍称结合vector quantization算法生成高质量的色彩范围，用脚本同样可以处理批量图片压缩。其支持命令行和源码库形式使用，这里介绍的是使用命令行方式：

1.首先，在 [pngquant官网](https://pngquant.org/)  下载适合windows系统的包：

![](http://i.imgur.com/niSwhMV.png)

下载下来的包含两个预处理的bat命令和pngquant.exe：

![](http://i.imgur.com/AZyrNsD.png)

因为我们自己编写python批处理脚本，所以这里只需要用到压缩的pngquant.exe工具；

2.现在就可以开始编写python脚本了，贴上核心代码：

	import subprocess
	#需修改为你自己本地pnguant.exe路径
	PNGQUANT_PATH = r'Your path\pngquant.exe'
	...
	cmd_command = '"{0}" 256 -s1 --force --quality=50-50 "{1}" -o "{2}"'.format(PNGQUANT_PATH, "unoptimized.png", "optimized.png")
	#执行命令
    p = subprocess.Popen(cmd_command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
    retval = p.wait()

以上就是对pngquant压缩代码的介绍，经测试发现，这种压缩方式有的情况下会出现压缩后的大小比压缩前还要大（压缩率为负数），笔者尝试调整--quality参数从20-100均不能解决问题（当然，并不能为了大小而影响图片质量，笔者认为参数quality在50已经是设置的最低值，这里只是做测试），暂时没找到有效的参数设置去解决，如果读者找到原因，希望可以分享下方法。

#Pillow (PIL Fork)

Pillow是python内置的PIL（Python Imaging Library）的一个分支，当初使用这个库主要是因为其支持python 3.0+版本，主要是对图片进行等比例缩放。后面做了测试发现，Pillow支持对png图片的压缩还是蛮可观的（比Tinypng逊色点），下面就简单介绍下其使用：[Pillow官网](https://pillow.readthedocs.io/en/3.3.x/) 

1.安装Pillow的python插件：

***注意，如果你电脑已经安装过python的 PIL库，请先卸掉再安装Pillow***

![](http://i.imgur.com/sJppxxf.png)

	pip install Pillow

或者<br>
	
	easy_install Pillow

2.现在就可以开始编写python脚本了，贴上核心代码：

	from PIL import Image
	...
	im = Image.open("unoptimized.png")
	if file.endswith(".png"):
		im = im.convert('P')
    im.save("optimized.png",optimize=True)

可以看到，上面做了一个图片是否为png的判断，主要是Pillow对png图片压缩能做到更好的处理。

#总结：

针对以上三种的图片压缩工具，如果你有足够多的API key或者图片只在最终发包使用前才压缩，优先考虑Tinypng，因为这个是压缩效果最好；而次之选择考虑Pillow，其较为接近Tinypng的压缩效果和出现压缩率为负数的图片几率数少；鉴于笔者没有找到有效解决pngquant出现压缩率为负数较多情况下的出现，所以笔者尽可能会减少使用。***特别注意的是，不要对同一张图片进行多次压缩，这样必然会影响图片的质量，得不偿失。***

源码参考：[https://github.com/hellokugo/imgCompress](https://github.com/hellokugo/imgCompress) 