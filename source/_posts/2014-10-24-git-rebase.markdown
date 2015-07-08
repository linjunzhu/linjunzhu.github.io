---
layout: post
title: "Git -- rebase 篇"
date: 2014-10-24 22:12:33 +0800
comments: true
categories: 
---
##git merge 和 git rebase

两者都是合并分支的功能，但是区别在于:

假如合并前是这样的：

```
D---E master
     /
A---B---C---F origin/master
```

使用merge后

```
      D--------E  
     /          \
A---B---C---F----G   master, origin/master
```

使用rebase后，就不会有G这个结点

```
A---B---C---F---D'---E'   master, origin/master
```

####Tips

1、D’, E’ 的 commit SHA 序號跟本來 D, E 是不同的，因為算是砍掉重新 commit 了。

2、因为使用`rebase`算是砍掉 D E 重新commit ,那么这里就可能会造成两次冲突，需要修改两次。而merge只需要修改一次

3、因此如果是小规模改动，冲突不会太大的话，建议使用rebase,否则使用merge。

使用reabse的好处是可以让分支不会那么乱，呈线性。


####进行合并
```ruby
两条分支： dev 和 master，此时 dev 需要合并回 master

# rebase 方式
$(dev) git rebase master
$(dev) git checkout master
$(master) git merge dev

# merge 方式
$(master) git merge dev

# rebase 方式，此时dev和master的指针都是同时指向最新的commit点
# 但是 origin/dev 指针头消失了。因此
# 如果要继续使用 dev, 需要先 pull 一下，跟 远端仓库的 dev 最新 commit 点进行一次 merge ( 当然结果是一个空 commit,此时可以 push --force)
# 此时 dev 分支的 nerwork 图是：
# * b6fa04b (HEAD, master, dev, origin/master) dev01 ( 10 seconds ago )
# 原因见下面
```

####解决冲突
```
# 解决冲突后
-------------------

# merge 方式
$ git add .
$ git commit -am 'xx'
$ git push origin master

# rebase 方式
$ git add .
$ git rebase --continue
```
	

#### pull 时使用 rebase 方式
```ruby
$ git pull --rebase ( 适合同个分支 ）
```

#### --no-ff
因为两条分支在合并时，当第一条分支完全没做修改时，此时 Git 会用到`Fast forward`模式，将 HEAD 指向第二条分支的最新 commit 去，这时候我们可以用该参数来禁止`fast forward`模式。

`git merge --no-ff`这参数的作用跟`rebase`恰恰相反，是故意弄出一条分支线，表明某些 commit 是为了实现某个功能的。

适用于分支间的合并


###Tips
一般分支间的 rebase 合并使用场景是：

1、我自己需要实现某个功能，于是开个分支 new_menu，不提交到远程分支。开发完毕用 rebase 弄回 dev 分支

2、一旦分支提交到远程分支，最好不要使用 rebase 进行分支间的合并了，会造成混乱。

3、一般来说，使用 rebase 后的 new_menu 分支就当做废弃了，如果还需要重新使用的话，继续从 dev 分支 new 出一条分支来。

4、在实验中我发现，一旦在 dev 进行 rebase master,那么 origin/dev 的指向就消失了，那是因为 dev 的原先 commit 全部消失了，进行重新 commit，因此本地的 origin/dev 也没得指向了。所以才会造成要 push 时要先`pull`的现象，(也就是 master分支上的最新commit点,「此时dev也是指向最新commit点」，会跟远程服务器上的origin/dev最新commit点进行merge)因此我们可以用`git push --force` 强制 push 来解决。

如果是`pull`:
![](http://data-storage.qiniudn.com/2546D03D-A4B6-4765-9DF4-0154F62606DC.png)

![](http://data-storage.qiniudn.com/C142553B-FE45-42B3-BC10-92111A969C3F.png)

如果是`git push --force` 的话就会成为一条直线。
##整理 commit
在一个 branch 上开发一段时间后，commit 看起来会很杂乱，或者很多无谓的 commit 点，那么我们可以使用 rabase 来将多个 commit 合并成一个。

```ruby
$ git log

=>
a
b1
b2

=>要变成

a
b
```

```ruby
# 拿到 a 的 SHA-1 后
$ git rebase -i 49687a0a646954afdf3f4dae1f914ea793341ea2
```

```ruby
pick 033beb4 b1 
pick d426a8a b2
# Rebase 49687a0..d426a8a onto 49687a0
# pick = use commit 
# squash = use commit, but meld into previous commit
# pick 会执行该 commit, squash 则将该 commit 合并到前一个 commit
```

于是我们修改成：
```ruby
pick 033beb4 b1 
squash d426a8a b2

# 保存退出后会要求输入新的 commit 信息，后可通过 git log 查看。
```

###Tips
1、这功能的使用场景是 commit 还没有 push 到远程分支，一旦 push 到远程分支就不要用了。( 虽然可以用 --force 强制 push，视情况而定 )