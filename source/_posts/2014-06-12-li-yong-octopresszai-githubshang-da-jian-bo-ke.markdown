---
layout: post
title: "利用OCtopress在github上搭建博客"
date: 2014-06-12 14:13:44 +0800
comments: true
categories: 
---

###目录

1. 安装ruby
2. 安装Octpress
3. 配置Octopress
4. 部署Octopress
5. 开始写博客
6. 更换主题
7. 绑定域名
8. 回顾


####1、安装Ruby

Octopress需要Ruby环境，那么就来装下Ruby环境。

首先安装RVM，RVM是用来负责安装和管理Ruby的环境

```
	curl -L https://get.rvm.io | bash -s stable --ruby
```

接着安装Ruby

```
	sed -i -e 's/ftp\.ruby-lang\.org\/pub\/ruby/ruby\.taobao\.org\/mirrors\/ruby/g' ~/.rvm/config/db (切换rvm 的安装源，天朝的GFW，你懂的。)
	rvm install 2.1.0
	rvm use 2.1.0 --default
	
```
参考：<http://rvm.io/rvm/install>

####2、安装Octopress

首先，请确认你的机子已经安装了git。
git 安装之后，利用git命令将octopress从github上clone到本机。

```
	git clone git://github.com/imathis/octopress.git blogName
	cd blogName  
	gem install bundler (如无bundler,则需要install, 一般rails开发人员都已安装)
	bundle install

	rake install （安装Octopress）
```
参考：<http://octopress.org/docs/setup/>

####3、 部署Octopress

我们可以在github上托管我们的blog。
具体做法：

1. 在github创建仓库，按照username.github.io 命名即可（之前是username.github.com,不过由于一系列原因改成了.io），随后十分钟左右即可在浏览器中访问username.github.io。

配置好仓库后，我们需要利用Octopress的一个配置rake任务，来自动配置blog。

```
	$ rake setup_github.pages
```

上面的命令会做一系列操作

1. 会创建`_deploy`目录（该目录在 master 下)，该目录用来存放部署到master分支的内容（username.github.io 的内容是在master分支下的）。
2. 并且source/_post/ 目录是存放你所写的文章源码。
3. 期间会要求输入远程仓库的url。
4. 会自动切换到 source 分支，以后就直接在这分支上干活。
5. 注意，master 分支不要动，完全可以忽视它了。


初次部署项目

```
	$ rake generate (根据配置等生成所需要的静态文件，如：文章)
	$ rake deploy (将_deploy目录部署到远程master分支)
```

源码需要单独提交到git

```
	
	$ git add .
	$ git commit -am 'first commit'
	$ git push origin source

```

####4、配置Octopress

大多数只需要配置_config.yml 和 Rakefile文件即可，不过Rakefile文件跟博客部署有关，一般不会去动。
_config.yml是博客很重要的一个文件，根据需求开启第三方组件等等。另外，其中的URL需要填写github上的远程仓库地址。 

####5、开始写博客

新建一篇文章

```
	$rake new_post\['标题'\]
```
该操作会在_post目录下生成一个markdown文件，编辑该md文件写文章。

```
	$ rake generate (每次修改文章都需要执行)
	$ rake preview  (在本地预览所写的内容 <localhost:4000>)
	$ rake deploy   (部署)
	
	随后把源码push到 source 分支
```

####6、更换主题
到网上搜索主题，比如我现在使用的主题[Slash](http://zespia.tw/Octopress-Theme-Slash/index_tw.html)。

```
	$ cd blogname
	$ git clone git://github.com/tommy351/Octopress-Theme-Slash.git .themes/slash
	$ rake install\['slash'\]
	$ rake generate
```


####7、绑定域名
比如我有一个 `linjunzhu.me`这个域名，那要如何做才能当我访问`linjunzhu.github.io`会自动跳转到`linjunzhu.me`进行访问呢？

1. 在 Github 中的博客仓库中，添加一个 `CNAME` 文件，里面写明我们所跳转的域名 `linjunzhu.me`
2. 使用 DNS 解析服务，将我们的域名指向该地址。

不过，由于 `rake generate` 和 `rake deploy`每次都会重新生成整个`master`分支，建议第一步不要直接在 Github 上操作。

而是在 Source 分支下的 source 目录添加`CNAME`文件。

这样当运行`rake generate`时会将`source`目录整个复制到`public`目录，运行`rake deploy`则会将`public`目录整个复制到`_deploy`目录。

####8、回顾
当成功后，我们访问`linjunzhu.github.io`时，会自动跳往`linjunzhu.me`，这时 Chrome 会自动缓存起来，以后访问`linjunzhu.github.io`都会自动跳往后者（无论有无 CANME）。

因此，当时我`CNAME`文件丢失后，访问`linjunzhu.github.io`总会自动跳往后者并且显示`404`，搞了许久以为是博客搞乱了，其实是 Chrome 的原因。

有人会问，为什么我还需要在分支上创建`CNAME`文件，而不能直接用 DNS 解析服务，将域名指向`linjunzhu.github.io`不就完事了吗？

因为 Github 做了处理，只有域名是`linjunzhu.github.io`才会跳往至博客页面，而访问`linjunzhu.me`，域名不符，是不会的，因此才需要第一步创建`CNAME`