---
layout: post
title: "Rails 中的 Assets Pipeline 机制"
date: 2014-10-07 21:59:06 +0800
comments: true
categories: 
---
##前言
我以前一直认为，Rails 的预编译就是把所有 css 和 js 分别整合成一个文件，并且进行压缩等操作。

其实并不是，Rails 预编译的意思，就只是预先编译那些 scss, coffee 等文件为 css, js 的意思。

而给文件名加上 hash 值得，整合成一个文件，压缩等等所有操作，都是Rails 中一个叫做 `Assets Pipeline` 功能实现的

## Asset Pipeline
`Asset Pipeline` 提供了一个框架，用于连接、压缩 JavaScript 和 CSS 文件。还允许使用其他语言和预处理器编写 JavaScript 和 CSS，例如 CoffeeScript、Sass 和 ERB。

`Asset Pipeline`在Rails4中已经从框架中提取出来成为：`sprockets-rails` gem

`Asset Pipeline`有三个主要功能：
 
 1. 连接所有静态资源，分别成为一个js css文件
 2. 压缩静态资源
 3. 允许使用高级语言编写静态资源，如 sass coffee

Rails 4 会自动把 sass-rails、coffee-rails 和 uglifier 三个 gem 加入 Gemfile。Sprockets 使用这三个 gem 压缩静态资源：

```ruby
gem 'sass-rails'
gem 'uglifier'
gem 'coffee-rails'
```


默认情况下，在生产环境中，Rails 会把预先编译好的文件保存到 public/assets 文件夹中，网页服务器会把这些文件视为静态资源。在生产环境中，不会直接伺服 app/assets 文件夹中的文件。

默认编译的文件包括 application.js、application.css 以及 gem 中 app/assets 文件夹中的所有非 JS/CSS 文件（会自动加载所有图片）

```ruby
[ Proc.new { |path, fn| fn =~ /app\/assets/ && !%w(.js .css).include?(File.extname(path)) },
/application.(css|js)$/ ]
```
这个正则表达式表示最终要编译的文件

最终会编译成： application-adfea1231s.js ,   application-sfef234df.css， xxx-sdfsdfe.png  等

因此如果我们想要自己添加单独编译出来的文件的话：

```ruby
config.assets.precompile += ['admin.js', 'admin.css', 'swfObject.js']
```

注意，即便想添加 Sass 或 CoffeeScript 文件，也要把希望编译的文件名设为 .js 或 .css。（ 因为 sass 最终也是要解析成 .css 的）

##对 Bootstrap 遇到的坑
之前引入了 Bootstrap 后，预编译一直出现
```ruby
Sass::SyntaxError: Undefined variable: "$alert-padding"
```

之后折腾了许久，才发现我在 assets.rb 里设置了
```ruby
config.assets.precompile += [*.js]
config.assets.precompile += [*.css]
```

因为我把所有 js css 都单独编译了，而 bootstrap 有两份 css 文件，其中一份定义了 bootstrap 的变量，如: `$alert-padding`，而这个变量在第二个 css 文件中被引用，因此，当第二个文件单独编译时，就会发现找不到这个变量了。

解决方法：

1. 按需加入自己要的文件

2. 编译所有静态资源，但是去除bootstrap相关的( 这里还没去除 )
```ruby
# config/application.rb
config.assets.precompile << Proc.new do |path|
  if path =~ /\.(css|js)\z/
    full_path = Rails.application.assets.resolve(path).to_path
    app_assets_path = Rails.root.join('app', 'assets').to_path
    if full_path.starts_with? app_assets_path
      puts "including asset: " + full_path
      true
    else
      puts "excluding asset: " + full_path
      false
    end
  else
    false
  end
end
```

##静态资源优化

###1. 设置过期时间

编译好的静态资源存放在服务器的文件系统中，直接由网页服务器伺服。默认情况下，没有为这些文件设置一个很长的过期时间。为了能充分发挥指纹的作用，需要修改服务器的设置，添加相关的报头。

针对 Nginx
```ruby
location ~ ^/assets/ {
  expires 1y;
  add_header Cache-Control public;
 
  add_header ETag "";
  break;
}
```

###2. 使用 gzip 压缩
Sprockets 预编译文件时还会创建静态资源的 gzip 版本（.gz）。网页服务器一般使用中等压缩比例，不过因为预编译只发生一次，所以 Sprockets 会使用最大的压缩比例，尽量减少传输的数据大小。网页服务器可以设置成直接从硬盘伺服压缩版文件，无需直接压缩文件本身。

在 Nginx 中启动 gzip_static 模块后就能自动实现这一功能：
```ruby
location ~ ^/(assets)/  {
  root /path/to/public;
  gzip_static on; # to serve pre-gzipped version
  expires max;
  add_header Cache-Control public;
}
```

如果编译 Nginx 时加入了 gzip_static 模块，就能使用这个指令。Nginx 针对 Ubuntu/Debian 的安装包，以及 nginx-light 都会编译这个模块。否则就要手动编译：
```ruby
./configure --with-http_gzip_static_module ( 还没试过）
```

##本地预编译
为什么要在本地预编译静态文件呢？原因如下：

1. 可能无权限访问生产环境服务器的文件系统；
2. 可能要部署到多个服务器，避免重复编译；
3. 可能会经常部署，但静态资源很少改动；

不过有两点要注意：

1. 一定不能运行 Capistrano 部署任务来预编译静态资源；
2. 必须修改下面这个设置；

在 config/environments/development.rb 中加入下面这行代码：
```ruby
config.assets.prefix = "/dev-assets"
```

修改 prefix 后，在开发环境中 Sprockets 会使用其他的 URL 伺服静态资源，把请求都交给 Sprockets 处理。但在生产环境中 prefix 仍是 /assets。如果没作上述修改，在生产环境中还是会从 /assets 伺服静态资源，但是除非再次编译，否则看不到文件的变化。

但为什么部署的时候不会再次编译呢？明明文件内容都发生变化了，奇怪？

之前预编译后，public下有/assets文件夹（production环境），但是请求的静态资源却全部都是 development 下的请求方式，（不带hash值的文件名)，奇怪- - 以后有空好好研究下。
