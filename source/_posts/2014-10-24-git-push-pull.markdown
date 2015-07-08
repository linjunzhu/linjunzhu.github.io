---
layout: post
title: "Git -- Push 和 Pull 篇"
date: 2014-10-24 21:12:33 +0800
comments: true
categories: 
---

##git push

####将本地仓库的更新推送到远程仓库 remote

`git push [remote-name] [branch-name]`

`git push origin master`

```ruby
# 如果当前分支跟远程分支有 tracking 关系，那么将会自动推送当前分支到对应的远程分支
git push origin 

# 如果主机名只有一个 origin，那么连 origin 都可以省略
git push
```
####查看 tracking 关系：
```ruby
$ cat ./.git/config

# 此时没有 tracking 关系
==>
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git@github.com:linjunzhu/test_branch.git
	fetch = +refs/heads/*:refs/remotes/origin/*
```

####添加 tracking 关系
```ruby
git push -u origin dev  (这句同时指定了 origin 为默认主机)
或者
git branch -u dev origin/dev
或者
git push --set-upstream origin master
```

####查看 tracking 关系
```ruby
$ cat ./.git/config

# 此时有 tracking 关系
==>
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git@github.com:linjunzhu/test_branch.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

## git pull
###拉取更新
将远程仓库的更新拉取到本地的「远程仓库」并且进行「merge」操作
```ruby
$ git pull origin master

# 相当于两步操作
$ git fetch origin master
$ git merge origin/master
```

###Tips
如果当前分支跟远程分支是没有 tracking 关系的，那么执行`git pull`后，会 download 远程仓库的所有分支代码到本地的 origin/分支上，但是不会合并，相当于 `git fetch` 命令一样。



##git pull 和 git fetch 的使用区别

1.  `git pull origin master`: 意思是从远程端下载最新版本到当前分支，并且自动合并

2.  `git fetch origin master`: 意思是从远程端下载最新版本到当前分支，但是并不合并。因此如果是git fetch 的话，就需要做两步操作。


```
	git fetch origin master
	
	git log -p master origin/master   (查看修改内容)
	
	git  merge orgin/master	(合并分支)
	
	或者进行 rebase
	
	下章讲解
```


## Tips

####一、`不带任何参数的 git push`有两种模式

1. `matching`模式，会推送所有`对应远程分支`的`本地分支`。
2. `simple`模式，只推送当前分支 (如果有 tracking 关系的话)

默认没有设置，需要打下命令
```ruby
git config --global push.default simple
```


####二、关于`push`and`pull`
一般如果没把握，最好写全。
```ruby
git push origin dev
git pull origin dev
```

####三、切换远程分支
当小伙伴的新建分支 push 到远程分支，我们也想要拉取这条分支
```ruby
# 如果当前分支没有tracking关系，那么git fetch 默认拉取所有分支
git fetch origin test_branch 
git checkout test_branch
```