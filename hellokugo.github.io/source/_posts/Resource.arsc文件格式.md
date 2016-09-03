title: Resource.arsc文件格式
date: 2016/6/14 17：53

categories:
- Mobile
tags:
- android
---

# 前言 #
在研究资源混淆的过程中了解到微信的资源混淆工具--[AndResGuard](https://github.com/shwenzhang/AndResGuard)，这个工具的强大之处不仅仅在于混淆了冗余的资源文件名称和明显减少apk的大小（这是官方说明，小弟小试了下效果不明显，有待考证），同时不涉及生成apk的编译流程，只需要输入apk和一些命令参数即可生成所需的混淆资源后的apk。<!-- more -->
（美团也有一套混淆资源的方法，介绍是修改AAPT处理资源文件相关的源码，牛逼哄哄的样子：[没错，我是链接](http://tech.meituan.com/mt-android-resource-obfuscation.html)）。Anyway，对于追求完美的技术人员而言，并不能满足于一个现成的工具，了解其中原理加以修改才可以灵活运用。AndResGuard主要是通过修改resources.arsc文件来打到混淆的目的，因此，本文主要是对resources.arsc的文件格式进行阐述。
至于详细的实现方案和流程可参考[AndResGuard原理](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd)

# 数据结构总体介绍 #

resources.arsc文件格式是由一系列的chunk构成。总体而言，可以大致分为五个chunk，分别是TYPETABLE，TYPEPACKAGE，TYPE_STRING， TYPETYPE和TYPESPEC；而这些chunk组成的结构就是资源索引表头部+字符串资源池+N个Package数据块。贴上一张传烂的图：

![](http://i.imgur.com/UVU3hWW.png)

是的，看完这图我也是懵逼的。没关系，下面作一个详细点的介绍。

# 头部信息 #

在介绍每个chunk之前，先说明下每个chunk均包含一个ResChunk_header的头部信息，具体的数据定义如下：

	struct ResChunk_header
	{
    // 是当前这个chunk的类型
    uint16_t type;

    // 是当前这个chunk的头部大小
    uint16_t headerSize;

    // 是当前这个chunk的大小
    uint32_t size;
	};

从上面的结构图可以看到，每一个chunk均是以ResChunk_header作为开头，在下面的介绍中，将会跳过。

# 资源索引表头部（TYPETABLE） #

![](http://i.imgur.com/K76gXh0.png)

可以看到，除了头部信息，还有一个package数的数据，一个apk可能包含多个资源包，这里就是指被编译的资源包个数。具体体现在有多少个TYPEPACKAGE。

# 字符串资源池（TYPE_STRING） #

![](http://i.imgur.com/zWKmKf3.png)

紧接着资源索引表头部的就是字符串资源池，这个chunk包含了所有package里定义的字符串。这个chunk的数据结构比较重要，在后面的TYPEPACKAGE里也包含着这一种chunk的数据结构，同时也是AndResGuard其中做修改数据的位置。下面简略地看下其数据结构介绍：

### ResStringPool_header ###
header：标准的Chunk头部信息结构</br>
stringCount：字符串的个数</br>
styleCount：字符串样式的个数</br>
flags：字符串的属性,可取值包括0x000(UTF-16),0x001(字符串经过排序)、0X100(UTF-8)和他们的组合值</br>
stringStart：字符串内容块相对于其头部的距离</br>
stylesStart：字符串样式块相对于其头部的距离

### 两个Array和两个Content ###
紧接着ResStringPool_header就是字符串偏移数组和字符串样式偏移数组，字符串偏移数组每个元素记录着每个字符串相对于StringContent（图中数据‘字符串’）开始的索引；同样的，字符串样式偏移数组每个元素记录着每个字符串相对于StyleContent（图中数据‘style’）开始的索引。

整个字符串资源区的结构示意图：

![](http://i.imgur.com/pV5kgSm.png)


# Package数据块（TYPEPACKAGE） #

在资源索引表头部中指定了package数目，所以，在这里开始，package数据块（包含TYPETYPE和TYPESPEC）是一个紧接着一个（假如有多个package）。</br>
下面，先介绍下TYPEPACKAGE这个chunk的数据结构：

![](http://i.imgur.com/p0g4EWp.png)

### ResTable_package ###
header：Chunk的头部信息数据结构</br>
id：包的ID,等于Package Id,一般用户包的值Package Id为0X7F,系统资源包的Package Id为0X01</br>
name：包名</br>
typeString：类型字符串资源池相对头部的偏移</br>
lastPublicType：最后一个导出的Public类型字符串在类型字符串资源池中的索引，目前这个值设置为类型字符串资源池的元素个数。在解析的过程中没发现他的用途</br>
keyStrings：资源项名称字符串相对头部的偏移</br>
lastPublicKey：最后一个导出的Public资源项名称字符串在资源项名称字符串资源池中的索引，目前这个值设置为资源项名称字符串资源池的元素个数。在解析的过程中没发现他的用途

### Type String Pool & Key String Pool ###
紧接着ResTable_package的是两个与字符串资源池有相同结构的字符串资源池，区别在于这里的资源池只是包含当前package的资源。

Type String Pool（图中数据‘资源类型字符串池’）是一个package中用到的类型字符串，如：anim，id，layout；

Key String Pool（图中数据‘资源项名称字符串池’），比如string.xml内容如下：

	<?xml version="1.0" encoding="utf-8"?>
	<resources>
    <string name="app_name">Test</string>
    <string name="hello_world">Hello world!</string>
    <string name="action_settings">Settings</string>
	</resources>

那么这个pool就收集了app_name，hello_world   action_settings 这3个字符串。这个pool是AndResGuard作修改的一处位置。

整个Package数据块区的结构示意图：

![](http://i.imgur.com/Mhj3UOF.png)

可以看到，在紧接着Type String Pool和Key String Pool后是剩下的两个chunk，在这里Type Specification Trunk和Type Info Trunk分别对应 TYPESPEC和TYPETYPE，下面作详细介绍。

# 类型规范数据块（TYPESPEC） & 资源类型项数据块（TYPETYPE）#

先来看一段TYPESPEC的介绍：

	类型规范数据块用来描述资源项的配置差异性。通过这个差异性描述，我们就可以知道每一个资源项的配置状况。知道了一个资源项的配置状况之后，Android资源管理框架在检测到设备的配置信息发生变化之后，就可以知道是否需要重新加载该资源项。类型规范数据块是按照类型来组织的，也就是说，每一种类型都对应有一个类型规范数据块。

![](http://i.imgur.com/ERotuBl.png)

国际惯例，大概了解下其数据结构：

	header：Chunk的头部信息结构
	id：标识资源的Type ID,Type ID是指资源的类型ID。资源的类型有animator、anim、color、drawable、layout、menu、raw、string和xml等等若干种，每一种都会被赋予一个ID。
	res0：保留,始终为0
	res1：保留,始终为0
	entryCount：等于本类型的资源项个数,指名称相同的资源项的个数。

再来一段TYPETYPE的介绍：

	类型资源项数据块用来描述资源项的具体信息, 这样我们就可以知道每一个资源项的名称、值和配置等信息。 类型资源项数据同样是按照类型和配置来组织的,也就是说,一个具有n个配置的类型一共对应有n个类型资源项数据块。

![](http://i.imgur.com/VBljfg6.png)

TYPETYPE的数据结构：

	header：Chunk的头部信息结构
	id：标识资源的Type ID
	res0：保留,始终为0
	res1：保留,始终为0
	entryCount：等于本类型的资源项个数,指名称相同的资源项的个数。
	entriesStart：等于资源项数据块相对头部的偏移值。
	resConfig：指向一个ResTable_config,用来描述配置信息,地区,语言,分辨率等

注意下的是，这里两个chunk的id均是标识资源的id，这里是指类似drawable和anim等类型，其他字段在这里不做介绍，结合AndResGuard重点介绍的是紧跟TYPETYPE后的ResTable_entry数组，这个数组是指具体每个元素的配置解释，而这里也是AndResGuard需要做修改的其中两个地方。

说实话，看到两段描述的时候有点混乱，查阅资料和说明例子后以为理解了，结合AndResGuard源码尝试去理解更换两个chunk数据，又陷入了无限的懵逼状态。归根结底，是不能理解类型和配置是如何进行映射。毛爷爷说过，实践是检验真理的唯一标准，于是乎，小弟写了个简单的helloworld的demo，主要截上资源配置：

![](http://i.imgur.com/4AZ99ZF.png)

这里要留意两点，一是类型drawable有三种配置，分别是drawable-hdpi，drawable-mdpi和drawable-xhdpi，这三个配置里均有ic_launcher.png；第二点是，在drawable-mdpi文件下有test.png。

然后就开始解析里面的resources.arsc，主要结果截图如下：

图一：

![](http://i.imgur.com/qNT1t9t.png)

图二：

![](http://i.imgur.com/0TKtzCp.png)

关注图一，主要是说明两个TYPESPEC，以parse restype spec chunk...开始解析，第一块的type_name:attr，没有元素，所以
entryCount=0；第二块的type_name:drawable，留意下entryCount=2，有两个元素，所以对应有两个TYPETYPE，这里是以parse restype info chunk...开始解析，这个也很好理解，因为demo是有两张图片，分别是ic_launcher和test，对应的是drawable-mdpi文件下的文件描述；</br>

这时候关注下图二，这个是另外两个TYPETYPE chunk的截图，自然而然想到的是另外两个drawable的文件资源描述，图中每个圈是每一个entry，entry中的index是对应package chunk中资源项名称字符串池的字符串偏移值，解析出来的字符串时str:ic_launcher，value中的data其实是在 字符串资源池（TYPE_STRING）解析出来的数据，在内部也是通过一个偏移值去获取该值；图二还有一个细节，也是困扰小弟几天的问题，就是即使在drawable-mdpi文件下没有test.png，在其数据结构的trunk中的entryCount=2，而在具体的entry中datatype=TYPR_NULL,这就很好地理解了AndResGuard中替换资源名字的方法，保持每个resId的唯一性，同时也可以针对具体的配置进行文件名替换。</br>

至此，最后的两个chunk也介绍完毕。归结而言，就是每个资源类型对应一个TYPESPEC，而紧跟每一种类型的就是其类型所包含的配置，每个配置就是对应一个TYPETYPE。


# 关注AndResGuard #

大概了解了resources.arsc结构，去了解AndResGuard混淆资源的方案就应该更为清晰了，简单看下替换方案：

![](http://i.imgur.com/Z6EDzF0.png)

首先，我们其实修改的是资源文件名(a.png)和资源文件的路径名（r\a\a.png），这样对应修改的就是TYPEPACKAGE和TYPE_STRING（对应方案1和3），而最后我们介绍的TYPETYPE和TYPESPEC是分别有TYPEPACKAGE和TYPE_STRING的字符串偏移值，这样对应也需要做修改（对应方案4），当然，缩短了文件命名必然会对整个table的大小造成改变，所以要重新计算并写入新的resources.arsc（对应方案5），结合源码分析，可能会有更深的理解。</br>

由于本人没有对resources.arsc做整体的透彻理解，只是在AndResGuard项目基础上去选择性关注要修改的点，如果在分析过程中有出现纰漏或者错误的地方，希望指出。

参考资料：
[http://blog.csdn.net/jiangwei0910410003/article/details/50628894](http://blog.csdn.net/jiangwei0910410003/article/details/50628894)
[http://www.cnblogs.com/njxxdx/p/4189971.html](http://www.cnblogs.com/njxxdx/p/4189971.html)



