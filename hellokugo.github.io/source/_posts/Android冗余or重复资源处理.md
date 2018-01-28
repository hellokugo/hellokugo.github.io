title: Android冗余or重复资源处理
date: 2018/1/28 12：25

categories:
- Android
tags:
- Android，AndResguard
---


# 前言

项目组在优化apk包体大小时，有一项是删除项目中冗余or重复资源，这对于自家app资源文件处理是很简单的，只要找到对应的资源文件删除即可。但是在删除过程中，发现有一些资源文件并不是很好的处理：

- 集成第三方sdk的资源文件
- 不同module的冗余资源
- google 提供的v4包等也扫出冗余资源

针对上面几个场景扫出的资源，在编译前处理就显得无能为力。google提供的shrinkResources可以去除无用资源，但反编译发现，shrinkResources扫描出来的无用资源并没有删除，只是变成空文件（.xml）或变成67字节文件（.png）。有没有办法在生成apk后进行处理？笔者参考 [AndResGuard](https://github.com/shwenzhang/AndResGuard) 实现方式，在AndResGuard基础上实现了只输入apk即可真正删除冗余or无用资源。

# 思路

这里分别就冗余和无用资源的删除做下简要实现流程：

- ***冗余资源***

1. 扫描apk（zip格式）各文件的CRC
2. 筛选CRC相同对应的文件路径，即这些文件都是相同的，只需要保留扫描出来的第一个文件路径
3. 重新生成resource.arsc，修改StringBlock，具体即把重复的资源路径全部替换成扫描出来第一份文件的路径
4. 之后的几个chunk原原本本copy一次，重写整份resource.arsc的size
5. 删除重复资源

- ***无用资源***

1. AS gradle设置 shrinkResources true***（必须）***
2. 扫描apk（zip格式）各文件的CRC
3. shrinkResources 扫描出来的无用代码文件，都会替换成预定义资源，其CRC定义是：

![](https://github.com/hellokugo/markdownPic/blob/master/dupilcate.png)

4.删除上述三种CRC文件

## 核心代码

Talk is cheap，show me the code. 说明一点，笔者是在AndResGuard基础上开发，一是没必要重复造轮子，二是resource.arsc字节码文件处理要特别小心，特别是修改StringBlock要注意四个字节对齐问题，AndResGuard本身是支持这一点，所以就站在巨人肩膀上开发。


获取CRC：

	Utils.java
	method:getZipCrc(add)
	
	public static Map<String, String> getZipCrc(String apkPath) {
        Map<Long, String> resMap = new HashMap<>();
        Map<String, String> duplicateMap = new HashMap<>();
        try {
            ZipFile zipFile = new ZipFile(apkPath);
            Enumeration e = zipFile.entries();

            while (e.hasMoreElements()) {
                ZipEntry entry = (ZipEntry) e.nextElement();
                String entryName = entry.getName();
                long crc = entry.getCrc();
                if (entryName.startsWith("res/")) {
                    //record useless files
                    if ("2293408688".equals(Long.toString(crc)) ||
                        "289995143".equals(Long.toString(crc)) ||
                        "3622196803".equals(Long.toString(crc))) {
                        duplicateMap.put(entryName, "");
                        continue;
                    }
                    //record duplicate files
                    if (resMap.containsKey(crc)) {
                        duplicateMap.put(entryName, resMap.get(crc));
                    } else {
                        resMap.put(crc, entryName);
                    }
                }
            }
            zipFile.close();
            return duplicateMap;

        } catch (IOException ioe) {
            System.out.printf("Error opening zip file" + ioe);
            return null;
        }
    }
    
修改StringBlock：

	StringBlock.java
	method:writeTableNameStringBlock
	
	for (i = 0; i < stringCount; i++) {
            stringOffsets[i] = offset;
            String originStr = getString(i);
            //如果没有duplicatedFile,直接拷贝
            if (duplicatedFile == null || !duplicatedFile.containsKey(originStr)) {
                //需要区分是否是最后一项
                int copyLen = (i == (stringCount - 1)) ? (block.m_strings.length - block.m_stringOffsets[i]) : (block.m_stringOffsets[i + 1] - block.m_stringOffsets[i]);
                System.arraycopy(block.m_strings, block.m_stringOffsets[i], strings, offset, copyLen);
                offset += copyLen;
                totalSize += copyLen;
            } else {
                String name = duplicatedFile.get(originStr);
                //是无用资源，无需修改StringBlock
                if("".equals(name)) {
                    continue;
                }
                if (block.m_isUTF8) {
                    strings[offset++] = (byte) name.length();
                    strings[offset++] = (byte) name.length();
                    ......
                    
删除冗余or无用资源：

	ResourceApkBuilder.java
	method:generalUnsignApk
	
        for (File f : unzipFiles) {
            String name = f.getName();
            if (name.equals("resources.arsc")) {
                continue;
            } else if (name.equals("res")) {
                //删除重复文件
                if (duplicateMap != null && duplicateMap.size() > 0) {
                    for (String duplicatePath : duplicateMap.keySet()) {               
                        File duplicateFile = new File(tempOutDir, duplicatePath);          
                        if (duplicateFile.exists()) {
                            duplicateFile.delete();          
                        }
                    }
                }
                continue;
            } else if (name.equals(config.mMetaName)) {
                addNonSignatureFiles(collectFiles, f);
                continue;
            }
            collectFiles.add(f);
        }

# 最后

目前该jar应用在项目jenkins持续集成中，无需手动扫描并删除资源，而且解决之前因各种原因无法在编译前删除对应资源的问题。不过目前该jar仍需完善，像无用资源并没有在resource.arsc删除对应的item，没处理在打包过后插进META-INF的无用文件等等，这个可以根据项目的实际需要再做完善。