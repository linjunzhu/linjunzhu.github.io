---
layout: post
title: "如何使用websocket-rails"
date: 2014-07-02 22:43:33 +0800
comments: true
categories: 
---

#如何使用WebSocket-rails

####1、WebSocket简介
WebSocket是Html5才出的一种协议，可以实现双向通信。而http请求仅能实现单向通信。comet也可以模拟双向通信，不过效率较低，需要耗费大量的服务器资源。

####2、应用范围
可以作推送，聊天室等等

####3、开始动工

1. 将`websocket-rails` gem 加入 `gemfile`, and then excute `bundle`
2. Run the installation generator: `rails g websocket_rails:install`

####4、操作Controller

1.创建Controller

	class ChatController < WebsocketRails::BaseController
		def initialize_session
			可以在这里做一些初始操作
	
				end
	
```
2.设置路由
在`events.rb`文件中加入：
	
	WebsocketRails::EventMap.describe do
		namespace :tasks do
			subscribe :create, to: TaskController, with_method: :create
		end
	end

也就是声明可以访问的websocket路由


3.客户端发送请求

在JS文件中，编写代码：

	var task = {
		name: 'i am roller'
		age: 15
	}
	
	var dispatcher = new WebSocketRails('localhost:3000/websocket')
	
	#对tasks命名空间下的create方法(前面所定义的路由)发送请求，并且请求参数为task
	dispatcher.trigger('tasks.create', task)
	
4.服务端处理请求

在controller中 handle 这个事件
	
	class TaskController < WebsocketRails::BaseControler
		def create
			# task为事先创建好的model
			task = Task.new message
			if task.save
				send_message: create_success, task, namespace: :tasks
			else
				send_message: create_fail, task, namespace: :tasks
			end
		end
	end
	
5.服务端发送请求
上面的代码，`send_message` 表示对客户端触发 'create_success'事件，参数为task, 命名空间为tasks

6.客户端处理请求

	dispatcher.bind('tasks.create_success', function(task){
		console.log('successfully created' + task.name)
	})
	
7.服务端发送请求，客户端处理请求的第二种写法

在客户端：
	
	var success = function(task){
		console.log('something');
	}
	
	var failure = function(task){
		console.log('something');
	}
	
	# 直接将方法绑定，然后由服务端去触发
	dispatcher.trigger('tasks.create', task, success, failure);
	
在服务端：

	def create
  		task = Task.create message
  		if task.save
  		  trigger_success task
 		else
  		  trigger_failure task
  		end
	end			ils>