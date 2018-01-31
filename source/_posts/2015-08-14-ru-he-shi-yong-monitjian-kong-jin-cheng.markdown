---
layout: post
title: "如何使用monit监控进程"
date: 2015-08-14 12:27:29 +0800
comments: true
categories: 
---

## 前言

前阵子服务器上的 puma 偶尔会挂掉，puma 不像 unicorn 有 master 进程保护着。因此就需要用 monit 来监控进程，当 puma 挂掉时自动重启

## 安装
Ubuntu:
```ruby
aptitude install monit
```

其他系统的安装方式：
[官网 Wiki](https://mmonit.com/wiki/Monit/Installation)

## 基本操作

`monit status` : 查看 monit 监控的服务状态 ( 需要开启 web 服务)

`monit reload` : 重新加载配置文件

`monit -t` : 检测配置文件是否正确

`monit quit` : 杀掉进程

`monit -c /etc/monit/monitrc` : 启动 monit 

`monit start server_name` : 启动某个服务

其他的输入 `monit -h` 查看即可。

## 介绍

配置文件位于：`/etc/monit/monitrc`

打开 Web 服务，查看监控情况
```ruby
 set httpd port 2812 and
    use address 172.16.10.6
    allow localhost   # 允许本地访问
    allow 172.16.20.0/255.255.255.0 #允许本 IP 段访问
    allow admin:monit      # require user 'admin' with password 'monit'
 
```

说下配置文件默认配置
```ruby

<!-- 检查周期，单位为秒-->
set daemon 120

<!-- 日志文件位置 -->
set logfile /var/log/monit.log

<!-- 存储 monit 实例的唯一 ID-->
set idfile /var/lib/monit/id

<!-- 存放监控进程的情况 -->
set statefile /var/lib/monit/state

<!-- 设置邮件-->
set mailserver smtp.aa.com port 25 USERNAME "aa@aa.com" PASSWORD "123456"

# 如果没配置 email, 会自动丢弃 alert
# 该设置可以保存 alert
set eventqueue
      basedir /var/lib/monit/events
      slots 100


<!-- Email 格式 -->
set mail-format {
     from: monit@$HOST
  subject: monit alert --  $EVENT $SERVICE
  message: $EVENT Service $SERVICE
                Date:        $DATE
                Action:      $ACTION
                Host:        $HOST
                Description: $DESCRIPTION

           Your faithful employee,
           Monit
}

# 制定报警邮件的格式
set mail-format {
	from: aa@aa.com
	subject: $SERVICE $EVENT at $DATE
	message: Monit $ACTION $SERVICE at $DATE on $HOST: $DESCRIPTION.
}

<!-- Email 接收人-->
 set alert sysadm@foo.bar with reminder on 3 cycles 

<!-- 开启 http 服务，查看监控情况-->

set httpd port 2812 and
    use address 172.16.10.6  # only accept connection from localhost
    #allow localhost
    #allow 172.16.20.0/255.255.255.0
    #allow 10.0.0.0        # allow localhost to connect to the server and
    allow admin:monit      # require user 'admin' with password 'monit'
   # allow @monit           # allow users of group 'monit' to connect (rw)
   # allow @users readonly  # allow users of group 'users' to connect readonly

include /etc/monit/conf.d/*
```

我们会在在`monitrc` 文件中的末尾看到这句
```ruby
# 包含该文件夹下的配置
include /etc/monit/conf.d/*
```

因此我们可以给每个要监控的进程写个`conf`配置文件。

## 重载配置

```
# 检查语法
sudo monit -t
```

```
# 重载配置
sudo monit reload
```

## 监控硬盘

在`/etc/monit/conf.d`下新建文件`filesystem.conf`

```
check filesystem datafs with path /dev/vdb
	if space usage > 50% then alert
	group server
```



## 监控 Nginx

在`/etc/monit/conf.d`下新建文件`nginx.conf`
```ruby
check process nginx with pidfile /opt/nginx/logs/nginx.pid
  start program = "/etc/init.d/nginx start"
  stop program  = "/etc/init.d/nginx stop"
  group www-data
```


##监控 Puma
在`/etc/monit/conf.d`下新建文件`puma.conf`
```ruby
check process puma_glassx
  with pidfile "/mnt/glassx/shared/tmp/pids/puma.pid"
  start program = "/usr/bin/sudo -u deployer /bin/bash -c 'cd /mnt/glassx/current && ~/.rvm/bin/rvm ruby-2.1.2 do bundle exec puma -C /mnt/glassx/shared/puma.rb --daemon'"
  stop program = "/usr/bin/sudo -u deployer /bin/bash -c 'cd /mnt/glassx/current && ~/.rvm/bin/rvm ruby-2.1.2 do bundle exec pumactl -S /mnt/glassx/shared/tmp/pids/puma.state stop'"
```

当我们手动 kill 掉进程，过一会就会发现进程又自动启动了~

## 环境
monit 用的是最基础的环境，个人的环境变量是没加载的，因此如果要执行一些命令，但却找不到命令时，需要注意环境变量的问题。

```
# 这里使用 /bin/su - deploy ，以用户deploy的环境去执行后续命令
check process process_server
  with pidfile "xxxx.pid"
  start program = "/bin/su - deploy  -c 'ruby xxx"
  stop program = "/bin/su - deploy  -c 'ruby xxx'"
 if 5 restarts with in 10 cycles then timeout
```

##搭配 Capistrano
打开`capfile`
```ruby
# Load DSL and Setup Up Stages
require 'capistrano/setup'

require 'capistrano/deploy'
#
require 'capistrano/rvm'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
require 'capistrano/sidekiq'
# 打开 monit 的监控 tasks
require 'capistrano/sidekiq/monit'
require 'capistrano/puma'
# 打开 monit 的监控 tasks
require 'capistrano/puma/monit'

# Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

由于`sidekiq`会在跑`deploy`的时候就执行 monit 相关任务，并且需要`sudo`权限，很麻烦，所以我们可以手动关掉，等到部署完成后再执行任务上传 monit 脚本

打开`deploy.rb`
```ruby
set :sidekiq_default_hooks, false
```

跑完`deploy`后，执行以下两条命令
```ruby
# 将 monit 的监控配置上传到服务器
cap production puma:monit:config
cap production sidekiq:monit:config
```

这两条命令在执行时需要`sudo`权限，原因是调用了
```ruby
sudo mv /tmp/monit.conf /etc/monit/conf.d/puma_glassx.conf
sudo mv /tmp/monit.conf /etc/monit/conf.d/sidekiq_glassx.conf
```

因为我们只是上传有关 monit 的配置脚本而已，只需执行一次，因此我们在执行失败后，进入服务器手动输入这两条命令，即可完成。

## Monit 自启动

如果服务器重启了，也要自启动 monit 才行，我们可以使用 Ubuntu 的 upstart 工具

/etc/init/monit.conf

```ruby
 # This is an upstart script to keep monit running.
 # To install disable the old way of doing things:
 #
 #   /etc/init.d/monit stop && update-rc.d -f monit remove
 #
 # then put this script here:
 #
 #   /etc/init/monit.conf
 #
 # and reload upstart configuration:
 #
 #   initctl reload-configuration
 #
 # You can manually start and stop monit like this:
 # 
 # start monit
 # stop monit
 #

 description "Monit service manager"

 limit core unlimited unlimited

 start on runlevel [2345]
 stop on starting rc RUNLEVEL=[016]

 expect daemon
 respawn

 exec /usr/local/bin/monit -c /etc/monit/monitrc

 pre-stop exec /usr/local/bin/monit -c /etc/monit/monitrc quit

```

```shell
# 加载配置文件
initctl reload-configuration
# 启动 monit
start monit
```



## 详细资料

官网 WIKI：

[https://mmonit.com/wiki/Monit/ConfigurationExamples#NginX](https://mmonit.com/wiki/Monit/ConfigurationExamples#NginX)

[https://mmonit.com/monit/documentation/monit.html#CONFIGURATION-EXAMPLES](https://mmonit.com/monit/documentation/monit.html#CONFIGURATION-EXAMPLES)