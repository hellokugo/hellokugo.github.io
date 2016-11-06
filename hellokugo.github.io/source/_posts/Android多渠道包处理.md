title: Android多渠道包处理
date: 2016/11/6 2：21

categories:
- Android
tags:
- Android，，多渠道，pyqt
---

# 使用说明
众所周知，Android会在不同的应用市场或者渠道去发布自己的产品，然后根据不同渠道的反馈去做相应产品的后续版本。目前用得比较多的是像集成友盟在AndroidManifest的meta-data加入渠道信息然后在代码读取，我们前期业务的做法是打包过程中在assets目录下加入标识文件记录渠道信息，由于这样的做法是在apk签名前插入，换言之每处理一个渠道就要做一次签名打包的操作，如果处理多个渠道消耗的时间就需要非常长，这样的大前提下就要思考是否有更好的打渠道方式。<!-- more -->

# 处理方案

我们知道，apk其实就是一个zip格式的文件。在 [zip格式介绍](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)中我们发现，zip留给我们一个可扩展的comment字段，这个字段的添加修改并不会影响签名后的apk包。换言之，以往我们在打多渠道的时候是每打一个渠道就要签名一个apk，现在用这种方式可以是实现签名一个apk来打多个渠道包，而且这些apk包并不影响原本的签名，这样所消耗的时间肯定会大大减少。后续发现github上有人提供这个打包方式，大概的思路和[packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin) 是一致的，区别在于考虑到业务打渠道的时机基本是在第三方提审过后签名apk才打入，这时候尽可能让打渠道的工作交由相应的运营，所以做出一个可操作简单友好的“傻瓜式”GUI，方便任何人都可以参与这项工作。

# 思路

这里要说明下，打渠道是用python语言，并且通过编写pyqt来实现界面化；读取渠道信息是在java语言层面，具体是直接读zip的字节码来获取。还有一点是，测试过程中发现通过zip comment打渠道方式在有的时候没有通过对apk重签名（apk重新签名会把这个字段抹去）也会出现读取不到渠道信息的情况，这样为了尽可能避免这种情况出现，我们参考了 [美团Android自动化之旅—生成渠道包](http://tech.meituan.com/mt-apk-packaging.html)，在签名后的apk插入空的META-INF/channel_file.txt做了规避处理，防止zip被破坏了导致渠道信息丢失而读取不到的问题。当然了，美团这种处理方案在apk重签名后也是会导致渠道信息丢失的问题。

# 实现

## 最终界面

![](http://i.imgur.com/CI6KNco.png)

![](http://i.imgur.com/wyepQoT.png)

整体而言，操作的界面还是比较简单的，主要是两个tab：第一个是查看渠道信息，这里面首先读取comment字段，没有渠道信息则读取META-INF目录下的标识渠道的空文件，如果也没有则友好提示；第二个是打渠道的tab，可以看到有一个Extra可选填入操作，这个字段是供业务扩展字段，可以为空，实现的规则也是comment和空文件标识同时写入，当然，channel是可填入多个渠道标识实现打包。

## 写入渠道信息

安装pyqt和使用这里就不做介绍了，主要是说说实现的核心代码：

 	with zipfile.ZipFile(tempApk,'a',zipfile.ZIP_DEFLATED) as myzip:
            log_utils.getLogger().debug('do insert empty_channel_file.....')
            # 遍历META-INF目录下文件，如果本来就存有渠道空文件，返回提示不能生成渠道包
            fileList = myzip.namelist()
            for f in fileList:
                if f.startswith('META-INF/channelId'):
                    log_utils.getLogger().info('channel file has exists: ' + f)
                    return ('Failed ! channel file has exists: ' + f)
            # 添加以渠道id命令的空文件到META-INF/文件下
            channel_file = "META-INF/channelId_{channel}".format(channel=symbol)
            if not os.path.exists("empty_file.txt"):
                log_utils.getLogger().debug('empty_file.txt is not exists..')
                return ('Failed ! empty_file.txt is not exists..')
            myzip.write("empty_file.txt", channel_file)

            log_utils.getLogger().debug('do change EOCD comment.....')
            myzip.comment = str(commentDict).encode('utf-8') # 修改comment字段
            myzip.close()
            targetName = os.path.join(out_dir,originName.split(".")[0] + '(' + symbol + ').apk')
            shutil.move(tempApk,targetName)

代码应该比较清晰，首先在apk的META-INF目录下添加空的渠道标识文件，然后再添加comment字段。

## 查看渠道信息

    with zipfile.ZipFile(signedApk,'r',zipfile.ZIP_DEFLATED) as myzip:
        tag = 'comment'
        info = str(myzip.comment,encoding = "utf-8")
        if not info: # 不存在comment字段，读meta-inf文件下空渠道信息
            fileList = myzip.namelist()
            for f in fileList:
                if f.startswith('META-INF/channelId'):
                    info = f.split('_',1)[1]
                    tag = 'empty_file'

        return info,tag

上述代码实现是先读取comment，没有则读取空文件标识，返回不同结果。


## java读取渠道信息

上述GUI是本地可供快速查询和打渠道的工具，在我们的业务代码里肯定也需要读取具体的渠道信息用于上报，上述思路中也说了，我们是直接读取zip的字节码来获取comment信息的，核心代码参考：

	private static void readComment(File file) {
		RandomAccessFile zipFile = null;
		try {
			zipFile = new RandomAccessFile(file, "r");

			long fileSize = zipFile.length();
			/* find the magic number in a reverse manner */
			for (long i = 1; fileSize - i >= 0; ++i) {
				zipFile.seek(fileSize - i);
				byte b = zipFile.readByte();
				if (b == 0x06) {
					zipFile.seek(fileSize - i - 3);
					/* check for magic "0x06054b50" in little endian */
					byte[] key = new byte[4];
					zipFile.readFully(key);
					if (key[0] != 0x50 || key[1] != 0x4b || key[2] != 0x05) {
						continue;
					}
					/* get the file comment size */
					byte[] tmp = new byte[18];
					zipFile.readFully(tmp);
					int commentSize = (tmp[16] & 0xff)
							| ((tmp[17] & 0xff) << 8);
					System.out.println("" + commentSize);
					if (commentSize > 0) {
						byte[] comment = new byte[commentSize];
						zipFile.readFully(comment);
						String commentStr = new String(comment, "UTF-8");
						JSONObject json = new JSONObject(commentStr);
						SYMBOL = json.getString("channelId");
					}
					break;
				}
			}
		}catch(IOException e){//读写EOCD出错，默认设置没有EOCD字段
			e.printStackTrace();
			//这里应该加入读取META-INF下空文件渠道标识的逻辑
			if（读取到空文件标识）{
				SYMBOL = "空文件标识"
			}else{
				SYMBOL = "";
			}
		}catch (JSONException e) {//有EOCD字段，但是没有channelId
			e.printStackTrace();
			SYMBOL = "";
		}  finally {
			if (zipFile != null) {
				try {
					zipFile.close();
				} catch (IOException ex) {
				}
			}
		}
	}

注意，这里在没有读到EOCD（即zip的comment字段）时，应该是会读取META-INF目录下时候存在空的标识渠道文件，这里没有做演示，读者可以自行补充。

[github上的源码和本地工具](https://github.com/hellokugo/packageMultiChannel)

