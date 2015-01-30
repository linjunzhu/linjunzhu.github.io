---
layout: post
title: "使用 Mosquitto"
date: 2015-01-30 16:04:40 +0800
comments: true
categories: 
---

##引言
这段时间团队需要跟 Google Glass 进行交互，因此要做一个推送机制。而 MQTT 协议的推送是当今最火热的一个。

Mosquitto 则是实现了 MQTT 协议的服务。

##安装
在 Ubuntu 下：
```ruby
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
sudo apt-get update
```

如果显示`apt-add-repository`没有识别，则可以：
```ruby
sudo apt-get install python-software-properties
```

##配置
安装完成后，所有配置都会在 `/etc/mosquitto`目录下。其中最重要的则是`mosquitto.conf` 文件，以下则是配置文件内容。

```ruby
# =================================================================
# General configuration
# =================================================================

# 客户端心跳的间隔时间
#retry_interval 20

# 系统状态的刷新时间
#sys_interval 10

# 系统资源的回收时间，0表示尽快处理
#store_clean_interval 10

# 服务进程的PID
#pid_file /var/run/mosquitto.pid

# 服务进程的系统用户
#user mosquitto

# 客户端心跳消息的最大并发数
#max_inflight_messages 10

# 客户端心跳消息缓存队列
#max_queued_messages 100

# 用于设置客户端长连接的过期时间，默认永不过期
#persistent_client_expiration

# =================================================================
# Default listener
# =================================================================

# 服务绑定的IP地址
#bind_address

# 服务绑定的端口号
#port 1883

# 允许的最大连接数，-1表示没有限制
#max_connections -1

# cafile：CA证书文件
# capath：CA证书目录
# certfile：PEM证书文件
# keyfile：PEM密钥文件
#cafile
#capath
#certfile
#keyfile

# 必须提供证书以保证数据安全性
#require_certificate false

# 若require_certificate值为true，use_identity_as_username也必须为true
#use_identity_as_username false

# 启用PSK（Pre-shared-key）支持
#psk_hint

# SSL/TSL加密算法，可以使用“openssl ciphers”命令获取
# as the output of that command.
#ciphers

# =================================================================
# Persistence
# =================================================================

# 消息自动保存的间隔时间
#autosave_interval 1800

# 消息自动保存功能的开关
#autosave_on_changes false

# 持久化功能的开关
persistence true

# 持久化DB文件
#persistence_file mosquitto.db

# 持久化DB文件目录
#persistence_location /var/lib/mosquitto/

# =================================================================
# Logging
# =================================================================

# 4种日志模式：stdout、stderr、syslog、topic
# none 则表示不记日志，此配置可以提升些许性能
log_dest none

#还有一种写入文件的模式
log_dest file '/var/lib/mosquitto/mosquitto.log'

# 选择日志的级别（可设置多项）
#log_type error
#log_type warning
#log_type notice
#log_type information
#log_type all

# 是否记录客户端连接信息
#connection_messages true

# 是否记录日志时间
#log_timestamp true

# =================================================================
# Security
# =================================================================

# 客户端ID的前缀限制，可用于保证安全性
#clientid_prefixes

# 允许匿名用户
#allow_anonymous true

# 用户/密码文件，默认格式：username:password
#password_file

# PSK格式密码文件，默认格式：identity:key
#psk_file

# pattern write sensor/%u/data
# ACL权限配置，常用语法如下：
# 用户限制：user <username>
# 话题限制：topic [read|write] <topic>
# 正则限制：pattern write sensor/%u/data
#acl_file

# =================================================================
# Bridges
# =================================================================

# 允许服务之间使用“桥接”模式（可用于分布式部署）
#connection <name>
#address <host>[:<port>]
#topic <topic> [[[out | in | both] qos-level] local-prefix remote-prefix]

# 设置桥接的客户端ID
#clientid

# 桥接断开时，是否清除远程服务器中的消息
#cleansession false

# 是否发布桥接的状态信息
#notifications true

# 设置桥接模式下，消息将会发布到的话题地址
# $SYS/broker/connection/<clientid>/state
#notification_topic

# 设置桥接的keepalive数值
#keepalive_interval 60

# 桥接模式，目前有三种：automatic、lazy、once
#start_type automatic

# 桥接模式automatic的超时时间
#restart_timeout 30

# 桥接模式lazy的超时时间
#idle_timeout 60

# 桥接客户端的用户名
#username

# 桥接客户端的密码
#password

# bridge_cafile：桥接客户端的CA证书文件
# bridge_capath：桥接客户端的CA证书目录
# bridge_certfile：桥接客户端的PEM证书文件
# bridge_keyfile：桥接客户端的PEM密钥文件
#bridge_cafile
#bridge_capath
#bridge_certfile
#bridge_keyfile

# 自己的配置可以放到以下目录中
include_dir /etc/mosquitto/conf.d
```


##权限
为了避免任何人都可以往服务器去 pull&push，我们需要设置权限。

使用自带工具进行设置账号：
```ruby
#add a count and then will ask you to enter a passwd
mosquitto_passwd -c /etc/mosquitto/passwd glassx

# delete
mosquitto_passwd -D /etc/mosquitto/passwd glassx
```

##启动
启动服务很简单，直接运行：
```ruby
mosquitto -c /etc/mosquitto/mosquitto.conf -d

#-c 表示加载指定配置文件
#-d 表示以后台服务运行
#-p 表示监听某个端口（默认为1883)
#-v 表示亢长的日志记录模式，相当于设置 log_type all
```

mosquitto 是个异步 IO 框架，经过测试可以处理 20000 个以上的客户端连接。

##推送
上面所创建的只是服务器， MQTT 有三个角色， 发布角色，服务器角色，消费角色。 此时我们便使用 Ruby 来实现发布角色。

我们会使用到一个叫做 `mqtt` 的 gem
```ruby
require 'rubygems'
require 'mqtt'

# Publish example
MQTT::Client.connect('mqtt://glassx:glassxpw@127.0.0.1') do |c|
  p c.publish('topic', 'message')
end

# # Subscribe example
# MQTT::Client.connect('test.mosquitto.org') do |c|
#   # If you pass a block to the get method, then it will loop
#   c.get('test') do |topic,message|
#     puts "#{topic}: #{message}"
#   end
# end
```

##接收

接收则是各个客户端的实现了，这里就不写了。

##引用
官网：[http://mosquitto.org/](http://mosquitto.org/)

简要教程：[http://blog.csdn.net/shagoo/article/details/7910598](http://blog.csdn.net/shagoo/article/details/7910598)