---
layout: post
title: "如何使用gitbook"
date: 2014-06-12 17:29:25 +0800
comments: true
categories: 
---

###GitBook
GitBook是一款利用git和markdown快速写出电子书的工具

####1、开始使用

首先

```
$ npm install gitbook -g
```
安装gitbook命令行工具, 该命令行工具是用Node.js所写。可以依靠该工具在本地生成预览

接着去官网下载编辑器 <https://www.gitbook.io>，并且在上面注册账号

####2、开始写书

![](https://camo.githubusercontent.com/c1b6c55fca8e171120ce1fd73afcee699cc2a98f/68747470733a2f2f7261772e6769746875622e636f6d2f476974626f6f6b494f2f676974626f6f6b2f6d61737465722f707265766965772e706e67)

在File菜单上点击 new book,创建一本新书。
然后在左边侧边栏右键创建capter,需要注意的是capter名不能为中文。要如何做呢？

1. 用英文名创建capter （其实就相当于在本地创建一个文件夹）
2. 右键修改capter名为中文

否则会出现错误。

然后就愉快的写书啦~~~~

####3、上传数据

Book => publish as => done

####4、协作写书

将book文件夹 share 出去即可

