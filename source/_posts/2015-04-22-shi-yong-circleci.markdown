---
layout: post
title: "使用Circleci"
date: 2015-04-22 12:11:21 +0800
comments: true
categories: 
---
这货棒棒的好用！

我们每次部署任务时，都会跑测试，测试通过再手动跑部署命令。而有了这货，就可以自动跑测试，跑完自动部署。

##第一步

先上[Circleci官网](https://circleci.com/)注册账号。默认有一个`free container`，如果该项目是`public`的，那么有四个`free container`。也就是你可以同时 hold 四个项目

##第二步

选择账号，选择仓库，建立容器container

##第三步

只要在 master 上面有新的 commit

就会自动直接跑了！！！
卧槽！！！
什么配置都不用啊！！！！

(当然只是build项目，跑测试，部署到哪里还是要自己添加命令的）

因为 Circleci 会自动检测你是什么项目，然后采取不同的默认配置进行跑你的项目。

##circle.yml

通过该配置文件你可以做很多事情，默认可以不用去动他，如果想添加自己的东西，可以写一些参数覆盖掉默认配置。

比如：
```ruby
machine:
  ruby:
    version: 2.1.2
  services:
    - redis
general:
  branches:
    only:
      - master
      - staging
deployment:
  production:
    branch: master
    commands:
      - bundle exec cap production deploy

```

指明了需要什么 branch，如何部署，该跑什么 services.默认的services有`mysql``Postgres``Redis``MongoDB`

##容器
每一个项目它都会给我们生成一个 VM，相当于一个虚拟机，因此我们还可以自己`ssh`上 VM，进行查看问题。