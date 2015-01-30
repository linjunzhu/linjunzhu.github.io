---
layout: post
title: "部署 RabbitMQ 经验"
date: 2015-01-30 16:04:25 +0800
comments: true
categories: 
---


## 引用

在项目中，将一些无需即时返回且耗时的操作提取出来，放入消息队列，进行异步处理，而这种异步处理的方式大大的节省了服务器的请求响应时间，从而提高了系统的吞吐量。

RabbitMQ 是一个在 AMQP 基础上完整的，可复用的企业消息系统。他遵循Mozilla Public License开源协议。

##安装

Ubuntu 下：

```ruby
deb http://www.rabbitmq.com/debian/ testing main

wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
sudo apt-key add rabbitmq-signing-key-public.asc

sudo apt-get install rabbitmq-server
```

```ruby
#/etc/default/rabbitmq-server
ulimit -n 1024
```


```ruby
# 开启服务
invoke-rc.d rabbitmq-server stop/start/etc.

# 开启 WEB 管理
rabbitmq-plugins enable rabbitmq_management
```

## 注意事项
1. 要自己添加 /etc/rabbitmq/rabbitmq.config 才行
2. 默认只有本机才能用 guest/guest 账号登陆。否则需要在rabbitmq.config 添加`[{rabbit, [{loopback_users, []}]}].`
3. rabbitq.config 的格式是：

```ruby
# 就算删除了里面的参数配置，也不能删除这个格式。
[
].
```

## 可能出现的错误
1. 总是提示 
```ruby
  ls: cannot access /etc/rabbitmq/rabbitmq.conf.d: No such file or directory
```

原因： 妈蛋是缺了目录啊！手动创建就可以了，之前一直以为是缺了文件，原来是目录，注意了 ` conf.d` `.d` 这个结尾的一般都是目录 

##懒
具体就不写了，不过有篇文章写得很好，看看就明白了。

[https://ruby-china.org/topics/22332](https://ruby-china.org/topics/22332)