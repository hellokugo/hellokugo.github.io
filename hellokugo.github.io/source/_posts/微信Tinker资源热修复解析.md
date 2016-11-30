title: 微信Tinker资源热修复解析
date: 2016/11/30 20：10

categories:
- Android
tags:
- Android，Tinker
---


# 前言

微信的热修复框架 [Tinker](https://github.com/Tencent/tinker) 开源已有一段时间了，同时他们也配套地开放了一个免费的 [补丁后台](http://www.tinkerpatch.com/) 供我们完整地体验了整个热修复流程，根据详细的文档和demo演示接入，目前的使用已经趋于稳定。相比当前比较热门的热修复框架，Tinker对于开发者而言是更加透明而且容易理解。<!-- more -->俗话说，工欲善其事必先利其器。既然Tinker的源码已经开放，就不能单单地停留在使用的层面上，遇到问题看文档也不能根本解决问题，所以，窥探Tinker内部实现原理，可以在接入过程出现问题较快定位，而且了解实现原理对于自身的学习也是有很大帮助。

# 说明

Tinker是支持代码、资源和lib（so）层面上的修复，个人认为，代码层面的修复采用微信自研的dexDiff才是Tinker的核心。由于笔者对dex的结构还没熟悉，所以这里不针对代码修复流程进行解析，[Tinker Dexdiff算法解析](https://www.zybuluo.com/dodola/note/554061) 一文对其中dexDiff解析还是比较详尽，前提条件下对dex结构和bytecode有了解才可以阅读下去。本文主要解析的是Tinker实现资源修复流程。

# 思路

Tinker文档上主要演示的是如何在AS通过gradle来构建生成patch包，同时也有介绍如何通过tinker-patch-cli.jar命令行去生成patch。下载Tinker的源码，工程结构：

![](http://i.imgur.com/xaIhAsb.png)

其中，工程tinker-build正是tinker-patch-cli.jar的构建代码：

![](http://i.imgur.com/4xPMJ6e.png)

CliMain.java正是入口类。

## 解析输入参数和tinker_config.xml

Talk is cheap，show me the code. 打开CliMain.java，入口main方法：

![](http://i.imgur.com/uxmG7tQ.png)

这里首先获取当前运行命令行的目录，以便后续没有指定output等做拼凑，然后开始解析输入参数。解析完一系列参数后：

![](http://i.imgur.com/uqjpzqd.png)

这里面做了三件事，首先，解析tinker_config.xml文件，这个文件在tinker-build的tool_output下有，当然，可以根据自己的需要去定制条件。由于本文主要介绍资源patch，所以只看资源条件:

![](http://i.imgur.com/HbzJQMl.png)

pattern指定要做patch的指定文件，ignoreChange指定你不想做patch的文件（即使前后发生改变也过滤不做patch），largeModSize指定做bsdiff的阈值。

解析完tinker_config.xml，定义输出文件的logger；然后开始做patch。

## tinkerPatch

tinkerPatch方法笔者认为可以分成三部分进行分析。第一部分是patch的核心，当然包含资源、代码和libs的patch；第二部分是生成配置文件，主要是对配置的tinkerId和一些patch信息的输出记录；第三部分则是生成patch包，具体代码：

![](http://i.imgur.com/h3gHyJp.png)

下面重点解析第一部分。

### AndroidManifest check，水很深

跟代码进到decoder.patch，看到这一行

	//check manifest change first
    manifestDecoder.patch(oldFile, newFile);

事实上，这简单的一句check manifest change，里面涉及要了解的东西不少，跟进去看：

![](http://i.imgur.com/MFdoKc5.png)

这里获取新旧apk的mainfest信息，定义AndroidParser对象，保存解析得到的四大组件和metaData标签等信息，还有通过各种转义解析得到整一份AndroidMainfest的string返回，他们对应的核心类均是继承自XmlStreamer的ApkMetaTranslator和XmlTranslator。

 - 如果要具体深入了解里面的解析原理，必须得对Android的resource.arsc和AndroidManifest.xml文件的二进制格式有一定了解，当然还涉及escapeXml10等转义知识，不然基本上读着读着就懵逼。By the way，你大可不用去了解其解析，因为笔者在了解其解析Manifest的ResourceId Chunk读取的/r_values.ini文件时发现，其实这里的解析用到的是github上一个开源项目，Tinker应该是在此基础上加以修改完善。
 - 有兴趣的同学可以去了解下：https://github.com/clearthesky/apk-parser。当然，你也可以直接在jcenter搜索Tinker定制的apk-parser-lib。

接下来在获取到的AndroidManifest解析信息后，对minSdkVersion进行判断处理，主要是对config输入的ignoreWarning进行风险规避，这个在Tinker的wiki上也有说明：

![](http://i.imgur.com/jxzFFey.png)

这里是针对第一种情况处理，后面对ignoreWarning的判断将不再说明。</br>
最后则是将新旧的AndroidMainfest的标签信息进行一一比对，如果ignoreWarning为false，则抛出异常。因为目前Tinker还不支持AndroidMainfest的热修复。

### 资源文件对比筛选

通过对AndroidMainfest的新旧对比之后，如果对比成功则开始进行资源文件的对比筛选：

	Files.walkFileTree(mNewApkDir.toPath(), new ApkFilesVisitor(config, mNewApkDir.toPath(), mOldApkDir.toPath(), dexPatchDecoder, soPatchDecoder, resPatchDecoder));

跟进ApkFilesVisitor的visitFile方法，跳过代码和libs的patch，直接到资源patch：

![](http://i.imgur.com/TZguPmv.png)

资源patch在ResDiffDecoder类中的patch方法中实现，其主要是做了以下工作:

1. 解压后的新旧apk中，oldFile对应的newFile不存在，则认为新的apk对oldFile资源删除，保存在deletedSet中（正如其注释上说的，这段代码永远不会走，所以没有任何实质作用，deletedSet下面会重新添加成员）；
2. 解压后的新旧apk中，newFile对应的oldFile不存在，则认为新的apk新添了newFile资源，保存在addedSet中，并且把newFile拷贝到tinker_result目录下；
3. 如果不存在上述两种情况，而且oldFile和newFile存在，对比oldFile和newFile的md5，相同则认为该资源文件没有做任何修改，直接返回；
4. 如果该文件不在指定的ignoreChangePattern清单里，并且不是AndroidMainfest.xml文件，走到这里：

![](http://i.imgur.com/moYNpBO.png)

假如该文件是resources.arsc，开始进行新旧resources.arsc的对比。如果之前有阅读过 [Resource.arsc文件格式](https://hellokugo.github.io/2016/09/03/Resource.arsc%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F/) 这篇文章，这里理解应该不难。要注意的是，每个chunk的headSize对应着不同的位置大小。最后则是重定义equals方法去比较各个chunk是否一致：

![](http://i.imgur.com/VmVebFg.png)	

5.如果在进行上述4步还没有return，说明新旧apk都存在该资源文件，并且有做修改：

![](http://i.imgur.com/sfvXFcX.png)

这里首先判断该文件是否大于指定的largeModSize，这个在config可以指定，如果是大于该值，用bsDiff算法得到newFile，并保存在largeModifiedSet中；如果小于该largeModSize，直接保存在modifiedSet，并拷贝到tinker_result目录下。

### 解压修改文件

其实上述生成步骤走完，tinker_result就是最终的patch包，而接下来要做的是配置文件的写入，大体上和客户端合patch流程是一致的。跟代码走到

	resPatchDecoder.onAllPatchesEnd();

具体处理在ResDiffDecoder类中的onAllPatchesEnd方法中实现，其主要是做了以下工作:

1. 对pattern的文件判断和返回的筛选patch文件判空进行处理；
2. 判断是否用gradle构建生成patch，这里分析命令行生成patch，所以里面逻辑跳过；
3. 重新对返回的patch保存进行筛选，具体是
 - 再次遍历解压后的新旧apk资源文件，筛选出删除的资源，保存在deletedSet（在上述patch分类资源时deletedSet其实一直为空，只有在这里才会被赋值）；
 - 返回的四个patch set把AndroidManfest.xml剔除，因为这个文件是不可能做修改；
 - 返回的四个patch set把指定的ignoreChange清单文件剔除；
4. 新建一个最终的资源合成包resources_out.zip，开始往该zip包添加最终资源（下面展开细说）：

	![](http://i.imgur.com/pqfC4ra.png)
5. 7zip压缩，这里分析没有设置该参数，跳过；
6. 记录基准包的resources.arsc获取Crc和新包的resources.arsc获取md5到res_meta.txt，用以在做合成patch时的安全校验；
7. 记录四个patch set的信息到res_meta.txt

### 合成resources_out.zip过程

在上述第四步合成resources_out.zip过程中，涉及到将一个压缩文件内的指定文件压到另外一个压缩包内。这里巧妙地根据zip格式去做筛选压缩。具体实现代码在com.tencent.tinker.build.util的genResOutputFile方法中：

![](http://i.imgur.com/EKi0Fmz.png)

跟进去TinkerZipFile，走到readCentralDir()方法。当时在看到这个名称的时候，仿佛有那么一点的熟悉。因为之前在搞zip的EOCD读取渠道号的时候也看到类似代码。果不其然，这里也有涉及到EOCD，但其只作为获取central directory的桥梁，大致流程是 

	获取EOCD->解析EOCD的centralDirOffset（CDO）和EntriesNum（num）->根据CDO去查找num个entry->保存entry的name和entry的键值对

其实这里获取的entry name，即是zip中每个文件名。关于zip格式，可以参考下其wiki介绍 [Zip (file format)](https://en.wikipedia.org/wiki/Zip_(file_format))。然后做文件筛选压到resources_out.zip（下面均简指目标zip）中，主要步骤如下：

1. 对于旧apk而言，只要是未被删除和修改的资源都压到目标zip，同时排除AndroidManifest.xml（目前这里面有bug，删除的资源也会被压到目标zip，是路径未被转义导致字符串不匹配，已提pr）；
2. 对旧apk中是否存在AndroidManifest.xml判空，有则直接压到目标zip，没有直接抛错；
3. 分别对修改和新增的筛选set从newZipFile（注意，这里不是指新apk，是指在patch筛选过程中对资源拷贝的tinker_result目录的tempZip）压指定资源到目标zip
4. 返回目标zip的md5值

走到这里，patch资源信息记录已经完成了，剩下的就是对新旧tinkerId和packageConfig的记录，最后就是把tinker_result目录打成patch apk。如果有阅读客户端合资源patch的源码，会发现和生成resources_out.zip的过程大同小异。

# 最后

总体而言，整个资源patch流程还是比较清晰明了，只是对相关的资源二进制文件进行读取，并没有想象中去修改resource.arsc实现资源patch。和之前分享的 [优化动态加载方案](https://hellokugo.github.io/2016/09/10/%E4%BC%98%E5%8C%96%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD%E6%96%B9%E6%A1%88/) 文章对资源进行动态加载的思想一样，都是沿用google官方的InstantRun全量替换资源的._ap文件（在这里就是一个patch）方案。So，RTFC永远是王道。