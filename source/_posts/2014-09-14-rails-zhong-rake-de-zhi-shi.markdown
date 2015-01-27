---
layout: post
title: "Rails 中 Rake 的知识"
date: 2014-09-14 00:39:14 +0800
comments: true
categories: 
---

###1、什么是Rack?

其实 Rails 就是一个 Rack app。

 Rack 为使用 Ruby 开发的网页程序提供了小型模块化，适应性极高的接口。Rack 尽量使用最简单的方式封装 HTTP 请求和响应，为服务器、框架和二者之间的软件（中间件）提供了统一的 API，只要调用一个简单的方法就能完成一切操作。	
 
当然， Rack 还包含了许多东西，如处理静态文件，缓存， Log 等。 

`这里的Rack可跟Rake不同`

###2、Rails 是如何启动的？
我们会在命令行打 `rails s`

这条命令其实是运行 app/rails， 而 rails 的内容为：

```ruby
#!/usr/bin/env ruby
APP_PATH = File.expand_path('../../config/application',  __FILE__)
require_relative '../config/boot'
require 'rails/commands'
```

而 s 参数 则会调用 Rails::Server 模块
```ruby
Rails::Server.new.tap do |server|
  require APP_PATH
  Dir.chdir(Rails.application.root)
  server.start
end
```

其中 Rails::Server 又是继承 Rake::Server

###3、逻辑
一切从Rack开始. 几乎所有的ruby web framework都是rack app. Rack对象响应call方法, 返回三元素的array, 分别是status code, header, content body. 只要你的项目符合以上三个要求, 就是一个合法的rack app. 可以运行它, 在浏览器访问, 看到完整的响应内容. 所以, 主流程即是request与response的地程. 我们所做的事情就是在中间加入一些自己的东西.

```ruby
require 'rack'
class HelloWorld
  def call(env)
    [200, {"Content-Type" => "text/html"}, ["Hello Rack!"]]
  end
end
Rack::Handler::Mongrel.run HelloWorld.new, :Port => 9292
```
一、 http request 到达 web server 后会即被rack封装, 而后你得到一个env对象. 它包含了客户的请求类型(get/post/put/delete/..), 请求的地址(env['PATH_INFO'], QUERY_STRING)等等.

1. 通过分析env, 我们知道客户的请求是指向哪个controller#action. 而后查看路由表我们的app能否响应此请求.

2. 路由表在新建app对象时通过routes方法来定义, 具体的做法是接受一个block, block内调用match, get, post等方法时, 生成路由规则加入路由表. 路由表里包含路径字符串的匹配正则, controller, action, params等等.

3. 参照上一条, 在路由规则中检查 env['PATH\_INFO'], 若匹配, 就知道了指向哪个controller的哪个action, 以及其params. 通过 ctrl_const = Object.const_get(params[:controller].capitalize)来得到相应的controller.

4. 通过 ctrl_const.new(env).call(params[:action]) 可以调用到相应的方法.

5. 到这一步, 已经初步描述了一个请求从客户端到服务器端并指向需要的controller#action的基本过程. 也即处理request的过程完成.

二、Response的过程:

1. 在ctrl_const.new上调用action后新的对象内就会拥有相应的实例变量. 这时通过Tiltgem, 按规则生成目标view的名字, 找到它, 而后render, render时将self作为scope传入. 至此view里可以调用action里所有的实例变量.Tilt.new(view).render self 得到了应该返回的html的内容.

2. 上一步render得到的只是Controller#action对应的view, 需要将它交给layout处理.同样的使用Tilt, 将上一步得到的partial view 放到block中提交过去. 这样layout中的<%= yield %>关键字生效. 至此, 得到完整的html内容.

3. Rack要求调用call后的返回的值是一个三值的array, 分别是[status code, head content, body].上一步得到的是html内容就是body部分.


####4、Rails中的中间件
```ruby
cd your/project/path
rake middleware

=>
use Rack::Sendfile
use ActionDispatch::Static
use #<ActiveSupport::Cache::Strategy::LocalCache::Middleware:0x007fd93e3aee58>
use Rack::Runtime
use Rack::MethodOverride
等等

Action Controller 的很多功能都以中间件的形式实现。下面解释各个中间件的作用。

Rack::Sendfile：设置服务器上的 X-Sendfile 报头。通过 config.action_dispatch.x_sendfile_header 选项设置。

ActionDispatch::Static：用来服务静态资源文件。如果选项 config.serve_static_assets 为 false，则禁用这个中间件。

Rack::Lock：把 env["rack.multithread"] 旗标设为 false，程序放入互斥锁中。

ActiveSupport::Cache::Strategy::LocalCache::Middleware：在内存中保存缓存，非线程安全。

Rack::Runtime：设置 X-Runtime 报头，即执行请求的时长，单位为秒。

Rack::MethodOverride：如果指定了 params[:_method] 参数，会覆盖所用的请求方法。这个中间件实现了 PUT 和 DELETE 方法。

ActionDispatch::RequestId：在响应中设置一个唯一的 X-Request-Id 报头，并启用 ActionDispatch::Request#uuid 方法。

Rails::Rack::Logger：请求开始时提醒日志，请求完成后写入日志。

ActionDispatch::ShowExceptions：补救程序抛出的所有异常，调用处理异常的程序，使用特定的格式显示给用户。

ActionDispatch::DebugExceptions：如果在本地开发，把异常写入日志，并显示一个调试页面。

ActionDispatch::RemoteIp：检查欺骗攻击的 IP。

ActionDispatch::Reloader：提供“准备”和“清理”回调，协助开发环境中的代码重新加载功能。

ActionDispatch::Callbacks：在处理请求之前调用“准备”回调。

ActiveRecord::Migration::CheckPending：检查是否有待运行的迁移，如果有就抛出 ActiveRecord::PendingMigrationError 异常。

ActiveRecord::ConnectionAdapters::ConnectionManagement：请求处理完成后，清理活跃的连接，除非在发起请求的环境中把 rack.test 设为 true。

ActiveRecord::QueryCache：启用 Active Record 查询缓存。

ActionDispatch::Cookies：设置请求的 cookies。

ActionDispatch::Session::CookieStore：负责把会话存储在 cookies 中。

ActionDispatch::Flash：设置 Flash 消息的键。只有设定了 config.action_controller.session_store 选项时才可用。

ActionDispatch::ParamsParser：把请求中的参数出入 params。

ActionDispatch::Head：把 HEAD 请求转换成 GET 请求，并处理。

Rack::ConditionalGet：添加对“条件 GET”的支持，如果页面未修改，就不响应。

Rack::ETag：为所有字符串类型的主体添加 ETags 报头。ETags 用来验证缓存。
```


[https://ruby-china.org/topics/21517](https://ruby-china.org/topics/21517)

[http://wp.xdite.net/?p=1557](http://wp.xdite.net/?p=1557)