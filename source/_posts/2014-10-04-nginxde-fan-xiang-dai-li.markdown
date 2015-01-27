---
layout: post
title: "Nginx的反向代理"
date: 2014-10-04 18:28:48 +0800
comments: true
categories: 
---

最近需要在微信、Google Glass 上开发，需要两者服务器上的回调信息，因此需要正式服务器来让两者回调，但是这样的话我总需要在服务器上调试，是非常麻烦的。因此反向代理就正中红心。

###1、什么是反向代理？
反向代理（Reverse Proxy）方式是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。

###2、使用方式

WEB服务器： Nginx

####步骤一
用自己熟悉的编辑器打开 Nginx 的配置文件 `nginx.conf` / `/etc/nginx/sites-available/default `

开始配置：

```ruby
server {
    listen 80;
    server_name linjunzhu.com;
    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
    }

```

其中`proxy_pass`为反向代理地址，意思是，当接受到 `80`端口的`linjunzhu.com`地址时，就自动将请求转发给服务器本地的`9090`端口

####步骤二
打开本地的 shell 工具
```ruby
ssh deploy@服务器ip -R 9090:127.0.0.1:3000
```

这样本地的`3000`端口就会跟服务器的`9090`端口进行绑定，当服务器转发请求到服务器本地的`9090`端口时，又会将请求转发给本地的`3000`端口

完成，Enjoy it : )