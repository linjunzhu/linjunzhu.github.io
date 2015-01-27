---
layout: post
title: "使用 rsync"
date: 2014-12-26 15:00:40 +0800
comments: true
categories: 
---
#使用rsync

##1. 服务器端
`/etc/rsyncd.conf`
```bash
uid = deploy
gid = deploy
use chroot = no
max connections = 4
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.lock
log file = /var/log/rsyncd.log

[test]
path = /var/test
ignore errors
read only = true
list = false
auth users = fuck
secrets file = /etc/backserver.pas
```	

`/etc/backserver.pas` （ 需要设置权限400）
```bash
hello:world   (该账号密码最好不跟服务器一样）
```

`usr/bin/rsync --daemon` 启动服务

`echo "/usr/local/rsync/bin/rsync --daemon" >> /etc/rc.local` 开机自启动

##2. 客户端

###2.1 同步方式

####2.1.1. 第一种方式是 服务器–客户端方式。
这种方式，服务器启动 daemon 守护线程，监听端口 873，并配置需要同步的模块，然后客户端连接 873 端口，认证并同步。

其中，同步所使用的账号密码是 rsync  单独配置的，与系统无关。

服务端运行rsync进程在daemon模式下， 客户端是普通的rsync进程。

####2.1.2. 使用 ssh 方式
本机 rsync 进程 直接通过 ssh 通道连接到远程， 并在远程ssh通道执行命令


两者是走不同协议，不同端口的，因此第一种方式服务器是不需要启动 rsync 服务的，当然还是需要安装这个程序

###2.2 使用账号密码登录（此时服务器需要启动 rsync --daemon 服务）

```bash
USER@HOST::MODE  # mode 是服务器设置好的模块名，这里不能用路径,注意两个冒号
```

`/etc/rsync_client.pas`
```bash
glassx # 只需要配置连接时使用的密码即可，必须与A服务器上定义的密码相同.
```

`chmod 600 /etc/rsync_client.pas`

`usr/bin/rsync -vzrtopg --progress --delete --password-file=/etc/rsync_client.secrets  hello@192.168.10.240::test   /var/rsync/test`


跟在IP后的`test` 是指服务端配置的模块

###2.3 使用SSH （ 此时服务器不需要启动 rsync --daemon 服务）
```bash
USER@HOST:Folder # 注意只有一个冒号,rsync由此判断使用ssh通道。而不是直接连接远端的873端口。

# 使用了SSH后，就不能再用 :test 了，而是要跟纯路径。
```

`usr/bin/rsync -vzrtopg --progress --delete -e ssh deploy@192.168.10.240:/home/deploy/test   /home/deploy/test`

注意权限的问题，本地文件夹要有足够权限去同步文件（也不能在该命令前加 sudo, 这样用ssh时会错乱，如果用密码连接则可以）

###2.4 参考
http://blog.sina.com.cn/s/blog_544f183101013zlo.html

http://tech.huweishen.com/gongju/1529.html

http://www.cszhi.com/20120312/rsync-simple.html
