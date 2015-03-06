---
layout: post
title: "后台任务 VS 消息队列 -「译」"
date: 2014-12-26 14:56:25 +0800
comments: true
categories: 
---
# 后台任务 VS 消息队列

一件常见的事情，我看到工程师们认为消息队列跟后台任务是相等的。这是他们所忽略的：消息队列是一个后台任务的超集。所有的信息进程都在后台完成，但是后台任务并没有通过消息队列来完成。

拿一个简单的使用例子：当一个用户注册时，我想要去发一条欢迎 email，正常你会想在后台发送这封 Email，因此不影响用户的使用体验，但你真的需要去安装 ActiveMQ，RabbitMQ 或者 Resque 去做这些吗？显然不需要。

在建造一个复杂的系统时消息队列是一个基本的建筑模式。你的许多系统组件也许被不同团队所编写，但是他们用队列实现消息发送来通信，一个组件可以发送消息给另外一个组件说：“请发送这封邮件”。但是消息队列系统有它们的代价：它们是复杂的，因为他们被设计成你的分发系统的地基，他们必须被部署和监控，他们必须是可靠并且高可用的。

很多人安装消息队列去运行简单的后台程序，我认为并不需要这么复杂，我有一个简单的问题：“我是在让两个不同的子系统进行通信还是仅仅在衍生相同的工作？”几乎每一个网站都会立即面临注册邮件案例。想想在用户的浏览器投票结果中，执行一些操作可能需要30-60秒的时间，衍生出单独的线程去运行这些工程是完全充足且太过于简单的，这也是我 girl_friday 项目背后的原因，我想要一个简单并且可靠的方式去运行后台任务，而不需要复杂的消息队列系统。

## 总结

总的来说， 简单的任务用 sidekiq 等后台任务来处理即可。

MQ 适用于比较大型的，异构系统，多个系统间的相互通信。

MQ 是 sidekiq 的超集。

Sidekiq 背后的原理是将任务都放入队列，然后用多个线程去运行任务。

## Sidekiq 使用场景

比如发邮件，完成一些耗时的任务等。

代码大概是：

```ruby
# app/workers/hard_worker.rb
class HardWorker
  include Sidekiq::Worker
	
  # 这里系统会不断将任务压入队列，然后启动线程去执行任务
  def perform(name, count)
    puts 'Doing hard work'
  end
end
```

## MQ 使用场景
将一堆任务压入消息队列，然后让其他系统取出任务进行实现。

虽然我也可以将「发邮件」压入消息队列，然后在系统内取出进行发邮件，不过总归大材小用

代码大概是：

```ruby

# 发布者

class Publisher
  # In order to publish message we need a exchange name.
  # Note that RabbitMQ does not care about the payload -
  # we will be using JSON-encoded strings
  def self.publish(exchange, message = {})
    # grab the fanout exchange
    x = channel.fanout("blog.#{exchange}")
    # and simply publish message
    x.publish(message.to_json)
  end

  def self.channel
    @channel ||= connection.create_channel
  end

  # We are using default settings here
  # The `Bunny.new(...)` is a place to
  # put any specific RabbitMQ settings
  # like host or port
  def self.connection
    @connection ||= Bunny.new.tap do |c|
      c.start
    end
  end
end

# 消费者

# dashboard/app/workers/posts_worker.rb
class PostsWorker
  include Sneakers::Worker
  # This worker will connect to "dashboard.posts" queue
  # env is set to nil since by default the actuall queue name would be
  # "dashboard.posts_development"
  from_queue "dashboard.posts", env: nil

  # work method receives message payload in raw format
  # in our case it is JSON encoded string
  # which we can pass to RecentPosts service without
  # changes
  def work(raw_post)
    RecentPosts.push(raw_post)
    ack! # we need to let queue know that message was received
  end
end

然后还有 Exchange queue 绑定啊之类的
```

相信看完代码会对两者的区别更清晰点。