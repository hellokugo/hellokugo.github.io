# 前言
作为一名专业的技术码农，经验积累和技术分享是一项必不可少的软技能。俗话说，好记性不如烂笔头，在平时工作中遇到的各种奇葩问题和学习感悟用文档记录是个很好的习惯，同时也可以用做分享，让别人或者自己少走弯路。目前网上各种技术博客可谓百花齐放，诸如CSDN、51blog和不少公司定制的分享博客等，当然也有不少公司内部分享的.md记录文档。当然了，如果你是一名“异类”的程序猿，不满足于现有的博客框架，不妨考虑下本文介绍的Github pages+Hexo+Nodejs搭建个人blog。

# 概要说明

网上对于这种博客的搭建应该是有相当多的介绍了，[简单的搭建](http://cstsinghua.github.io/2016/06/16/Github%20pages+Hexo+Nodejs%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BAblog/#建站")或者换一个皮肤相信也难不到大家。但是大家在使用的过程中应该会发现，框架的搭建是依赖于当前的电脑，假如电脑要重装或者直接挂了要换一台新的，这样原本的记录岂不是全没了？我一直都相信，高手在人间，发现这个问题的早有人在，而解决的办法总比问题多，本文的介绍就是参考网上的配置做法做点补充：[参考文章](http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2/#more)

# 搭建流程

参考上面提供文章链接中的第四点 **优化部署与管理**，笔者在配置过程出现了几个问题导致一直配置失败，在这里分享下希望有同样问题的读者可以参考，过程大同小异：

1. 创建仓库，CrazyMilk.github.io；<br>
2. 创建两个分支：master 与 hexo；<br>
3. 设置hexo为默认分支（因为我们只需要手动管理这个分支上的Hexo网站文件）；<br>
4. 使用git clone git@github.com:hellokugo/hellokugo.github.io.git拷贝仓库；<br>
5. hellokugo.github.io文件夹下通过Git bash依次执行npm install hexo、hexo init、npm install 和 npm install hexo-deployer-git（此时当前分支应显示为hexo）;<br>

	**注意1：笔者在这里踩了坑，在执行hexo init的时候会把第二步clone下来的文件.git给删掉，导致在下面做代码提交步骤时一直报找不到.git文件错误，所以，在执行第五步或者执行hexo init前把.git文件放到上级目录（或者在执行完第五步再把.git文件放回来，这个没有做测试），这样就不会出现这样的错误了。这里可以用git branch来检查下当前分支会否为hexo**<br>

6. 修改_config.yml中的deploy参数，分支应为master（可用git branch命令来检查）；
7. 依次执行git add .、git commit -m “…”、git push origin hexo提交网站相关的文件；

	**注意2：这里最后一步git push origin hexo，对于还没习惯git工具的读者可能会直接按照这个命令输入，然后就抛出origin未定义的错误，这个如果有点git工具使用经验的读者来说，是应该在这命令前先定义origin，即git remote add origin git@github.com:hellokugo/hellokugo.github.io.git（这里是你的仓库github地址）**<br>

8. 执行hexo generate -d生成网站并部署到GitHub上。

	**注意3：有些读者可能走到这一步时就抛出FATE Deploy failed: git，这个问题在于执行第五步的最后一句命令npm install hexo-deployer-git没有把相关文件记录下来导致出错，不过这可能不是每个人都会出现，如果出现了，再执行npm install hexo-deployer-git --save,，然后再部署即可。**<br>

# 最后

感谢参考文章中笔者提供这种思路去巧妙解决电脑各种丢失数据而导致的问题。最后，真心想说一句，站在巨人的肩上，技术路上不孤单啊。