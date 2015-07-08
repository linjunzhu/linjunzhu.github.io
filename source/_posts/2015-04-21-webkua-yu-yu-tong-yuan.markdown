---
layout: post
title: "WEB跨域与同源"
date: 2015-04-21 20:08:09 +0800
comments: true
categories: 
---
这几天遇到`跨域`跟`同源`的问题。整理下知识点。

# 同源策略
浏览器有一个很重要的概念——同源策略(Same-Origin Policy)。所谓同源是指，域名，协议，端口相同。不同源的客户端脚(javascript、ActionScript)本在没明确授权的情况下，不能读写对方的资源。

如果Web世界没有同源策略，当你登录Gmail邮箱并打开另一个站点时，这个站点上的JavaScript就可以跨域读取你的Gmail邮箱数据，这样整个Web世界就无隐私可言了。

不同源举例：
```ruby
# 同源要求同域名同协议同端口
域名：
http://www.bigertech.com
http://bigertech.com
http://www.bigertech.cn

协议：
http、https

端口
127.0.0.1:3000、 127.0.0.1:3001
```

常见的：

我的 A 站点，发了个 AJAX 请求到 B 站点，会收到这样的错误
```ruby
No 'Access-Control-Allow-Origin' header is present on the requested resource
```

# 解决方法

1、在 Rails 中

```ruby
after_action :set_access_control_headers 

def set_access_control_header
  headers['Access-Control-Allow-Origin'] = '*'
  headers['Access-Control-Request-Method'] = '*'
end
```
当然这里偷懒指定了所有域名都允许跨域，一般来说是指定我们允许的域名

2、使用`JSONP`方式

JSONP(JSON with Padding)是一个非官方的协议，它允许在服务器端集成Script tags返回至客户端，通过javascript callback的形式实现跨域访问（这仅仅是JSONP简单的实现形式）

虽然浏览器有`同源策略`，但是`HTML`中的`img,script`等标签是不用遵循同源策略的， JSONP 则是利用了 script 这个特点。

服务端：
```ruby
# callback 是 rails 直接支持 JSONP 的参数
# hello 一般是客户端给过来的 callback 函数名
# `callback函数` 将在客户端进行执行 script
def index
   render json: {hello: :world}, callback: hello
end

# 关闭 csrf 防护
# 因为返回格式为 xxx()，属于危险字符script
# protect_from_forgery with: :exception

# 返回结果
/**/hello({"hello":"world"})

```

客户端：
```ruby
# 方式1
<script src="localhost:3001/students">

# 方式2 ( 使用 jquery )
function hello(){
  console.log("something");
}

$.ajax({
    type: 'GET',
    jsonp: 'callback函数名',
    dataType: "jsonp",
    url: "http://localhost:3001/students",
    success: function(data) {
       # 一旦返回将会自动执行hello()
    },
    error: function(XMLHttpRequest, textStatus) {
      console.log("error");
    }
});
```

#Tips
虽然说有`同源策略`的存在，站点 A 拿不到站点 B 的信息。但其实站点 B 已经处理完请求 response 回去了，只不过被站点 A 的浏览器给拦截了。

```ruby
# 站点 B，此时已经成功处理请求返回 response 了
Started GET "/students" for ::1 at 2015-04-21 20:32:32 +0800
Processing by StudentsController#index as */*
Completed 200 OK in 1ms (Views: 0.1ms | ActiveRecord: 0.0ms)
```

因此，如果你在后台代码中 / Shell 中进行访问请求的话：
```ruby
# 是成功返回信息的
$ curl http://localhost:3001/students

=>   /**/hello({"hello":"world"})%
```

有人说这样子那同源策略有什么用？不是一下子就被破解了么？

首先，`同源策略`是浏览器的行为。

其次，当你登录了网银后，打开`站点A`，此时`站点A`向`网银`发送一条`request`，注意，此时是会附带你`网银cookies`过去的，相当于你在`网银`自己发的`request`一样，如果没有`同源策略`，这时你的`网银`就相当于别人的了。。。（当然网银的安全程度远远大于此- -)

而如果你用后台进行发送请求的话，是不会附带任何`cookies`的。因此也就无效了。

注意 `postman`是可以成功请求的。（不知道postman做了什么，总之postman没有阻止跨域的情况)

## XSS 与 CSRF

说起安全，这两者可是 WEB 届大名鼎鼎的漏洞明星啊！

XSS :  跨站脚本攻击

CSRF : 跨站请求伪造攻击。

以下這篇文章写得非常好，值得一看。
[http://snoopyxdy.blog.163.com/blog/static/60117440201281294147873/](http://snoopyxdy.blog.163.com/blog/static/60117440201281294147873/)