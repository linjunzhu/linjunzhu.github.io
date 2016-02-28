---
layout: post
title: "Web 缓存 以及 Rails 缓存实践 -- 前端篇"
date: 2014-10-05 11:43:10 +0800
comments: true
categories: 
---
##1、前言
之前一直对缓存处于模模糊糊的概念，网上的许多文章也说得比较模糊，经过大量的实践以及搜索后，总结了下 Web  缓存以及在 Rails 中常用的缓存

##2、概览
我们平常所使用的浏览器，是有带缓存区的。

![](http://7jpo3s.com1.z0.glb.clouddn.com/web-cache-2.png)


##3、 Expires / Cache-Control 

两者功能差不多，但后者可供选择的参数更多，两者若同时存在，会被 cache-control 所覆盖
### Expires
用来表明该内容什么时候过期， HTTP/1.0 推出
```
# 表示在这个时间点之前资源都不会过期，可以直接从本地浏览器缓存区里拿
Expires:Tue, 03 May 2016 09:33:34 GMT
```
### Cache-Control

Cache-control 是 HTTP/1.1  推出的,有更多的参数可以选择

```
public: 共有缓存，可被缓存代理服务器缓存,比如CDN

private: 私有缓存，不能被共有缓存代理服务器缓存，可被用户的代理缓存如浏览器。

max-age：表示在这个时间范围内缓存是新鲜的无需更新。类似Expires时间，不过这个时间是相对的，而不是绝对的。也就是某次请求成功后多少秒内缓存是新鲜的。

no-cache：这里不是不缓存的意思，只是每次在使用缓存之前都强制发送请求给源服务器进行验证，检查文件有没改变

no-store：就是禁止缓存，不让浏览器保留缓存副本

must-revalidate：告诉浏览器，你这必须再次验证检查资源是否过期, 返回的代号就不是200而是304了。
```

需要注意的是，该参数 `request header` 与 `response header` 都可使用。但是总的来说，前者并不重要，重要且频繁使用的是后者 `response header`

### Request 内的 Cache-Control
除了 AJAX 可以来自定义这个 Cache-Control, 一般来说都是交给浏览器去控制。
```ruby
# 这里是直接告诉 代理缓存服务器 / 服务器，我要最新的副本 ( 然后再通过 ETag / If-None-Match 去判断是否从缓存区里拿 )
# 之前很疑惑，既然 response 都有 no-cache 了，那 request 设这个值的意义何在呢？
# 目的是告诉「代理缓存服务器」强制向「服务器」对资源进行重新验证，避免权限过期后，代理缓存服务器还是返回之前的资源副本。
cache-control: no-cache
```


### Response 内的 Cache-Control
```ruby

# 允许客户端进行缓存，并且，需要立即进行请求服务器验证资源是否过期
cache-control: no-cache
```

```ruby
# 表示 1000 秒内，浏览器需要这资源时，直接从缓存区内拿，而不用重新发请求
# 当过期时，发送「条件 GET 请求」
cache-control: max-age: 1000
```
```ruby
# 经常会看到这样写
cache-control: max-age: 0, must-revalidate

# 其实等同于
cache-control: no-cache
```


总而言之， Cache-Control 这个值主要是 response 来使用，request 可以不用去管它。


##4、条件 GET 请求

如果浏览器缓存中已经有一个 HTML 内容元素的副本，却不能保证该缓存是否有效，这时，一个条件 GET 请求就出现了。

![](http://data-storage.qiniudn.com/%E6%9D%A1%E4%BB%B6get%E8%AF%B7%E6%B1%82.png)

有两种方式判断资源是否需要过期：

浏览器会附带`If-None-Match`属性（内容id ) 或 `If-Modified-Since`（修改内容时间戳）

### ETag / If-None-Match

ETag 指响应内容的 MD5 值, 对应请求的 `If-None-Match`属性, 如果两者一致，则资源不过期

### Last-Modified / If-Modified-Since

`Last-Modified` 指响应内容的最后修改时间 ，对应请求的 `If-Modified-Since`，如果最后修改时间一致，则资源不过期

### 小结
ETag / Last-Modified 很多时候都会共同使用，但是浏览器会先以 `ETag` 为准，因为根据时间来判断是否过期，会有误差。


##5、 Web Server
静态文件的请求我们一般会用 Web Server 来 hold 住，比如 Nginx 

可以这样设置：
```ruby
location ~* /assets/ {
    # 开启压缩
    gzip_static on;
    # expires 过期时间为最长
    expires max;
    # 设置为 public,即 CDN 服务器也可以缓存
    add_header Cache_Control public;
}
```

PS：为了兼容，Nginx 如果设置了`expires max` 会自动往 cache-store 中塞`max-age`

##6、TIPS
1、如果已经打开网页，并且在地址栏按一次 enter, 那么所有缓存措施会生效，比如设置了 max-age 并且未过期的情况，会直接从缓存区拿数据

2、如果按浏览器的「刷新」按钮，会强制重新发送请求到服务器，但这时候 Etag / Last-Modifed 会派上用场

3、如果按浏览器的「强制刷新」，会强制重新发送请求到服务器，并且所有缓存失效。

4、服务器返回响应时，是先返回状态码，再返回响应体。

5、其实服务器还是执行了代码，生成了该有的响应，只不过浏览器知道资源未过期，因此把 响应体抛弃掉了。但总的来说还是节省了浏览器的时间。

6、条件 GET 请求中的 If-Modified-Since / ETag 是需要服务器支持的。

##7、总结
![](http://7jpo3s.com1.z0.glb.clouddn.com/web-cache-1.jpg)