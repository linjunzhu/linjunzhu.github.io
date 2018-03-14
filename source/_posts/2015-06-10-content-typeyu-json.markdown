---
layout: post
title: "content-type与json"
date: 2015-06-10 18:00:47 +0800
comments: true
categories: 
---

##前言
最近有两个 Android 同事在调用我写的 API 时，向我反映调用有问题，仔细一查，发现是客户端所设置的 Content-Type 与 Accept 有关系。所以写一下，后续有同事还有疑问，就可以丢这文章给他了~~

##Content-Type
该 header 头是用来告诉服务器，请求体是如何进行编码的，以便服务器进行相对应的解码。

一般来说，有如下几种:

1. application/x-www-form-urlencoded
2. multipart/form-data
3. application/json
4. application/xml
5. text/html
6. text/plain

等等

###application/x-www-form-urlencoded
这是我们最常见的 content-type 头了，我们使用浏览器原生 form 表单，提交时便会自动设置 content-type 为这个，其中提交的数据按照：` key1=val1&key2=val2`进行编码，key val 会进行 URL 转码

注意：content-type 头仅仅只是告诉服务器要如何解码，跟浏览器发送请求时对请求体如何编码无关。

也就是说，此时用 form 表单提交，但是 content-type 手动设置为 application/json，那么浏览器还是会以 form 形式对请求体编码。

###multipart/form-data
当我们需要上传文件时，就会用到这种方式。
```ruby
Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA

------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="text"

title
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="file"; filename="chrome.png"
Content-Type: image/png

PNG ... content of chrome.png ...
------WebKitFormBoundaryrGKCBY7qhFd3TrwA--
```
浏览器会以`multipart/form-data`形式进行编码，同时设置 content-type

###application/json
这种方式在当今 restful 盛行的年代，特别常用。

我们只需要将 json 格式的参数附带到请求体便是，同时手动设置 content-type 为 application/json ( jquery 中的 ajax,设置某个参数为 json即可，还有其他的，主要看框架是否会自动处理，否则就手动设置）


##Accept

该 header 头是告诉服务器，客户端需要什么格式的响应。

1. Accept : application/json
2. Accept : text/html


##要求服务器返回响应的格式

###一、
平常客户端设置该 Accept 参数，即可要求服务端返回响应类型的数据。

###二、
直接在 URL 尾部添加 `.json`，这种方式需要服务端的支持，告诉服务端，我需要`json`的响应

在 Rails 中，如果附带`.json`，会自动解析成一个参数 params[:format] = 'json'，之后便能通过代码针对不同解析返回不同数据

```ruby
# Rails 会自动处理 Accept 以及 .json 两种情况
respond_to do |format|
  format.html { redirect_to groups_url }
  format.json
end
```

###三、
在 Rails 中设置默认 format

```ruby
resources :users, defaults: {format: :json}
```

这样服务器就会默认有 format 为 json 的参数

那我同时设置了`Accept`以及`format`怎么办？

在 Rails 中，`format`的优先级高于`Accept`

##同事遇到的问题
1、调用接口时，反应接口返回了`500`，看了下，发现是服务器接收到的请求头为：content-type: application/json，但是请求体却是以`application/x-www-form-urlencoded`的形式进行编码的，因此两者不对称，解析出错。

2、调用接口时，返回的并不是 json 数据，check 后，发现调用接口时，没加 Accept: application/json，并且 URL 尾部没加`.json`，因此服务器就果断返回`text/html`对应的数据回去啦~~~


##需注意的点
如果没有手动指定 `Accept`头，一些 HTTP 请求包装库会手动给 `Accept` 头加上一些信息。如：

```
Accept: application/json,text/plain, */*;
```


这种情况，Rails 是不认前面的 `application/json`的，最终 `request.format` 只会返回 `text/html`.

原因如下：

```
 def format(view_path = [])
   formats.first || Mime::NullType.instance
 end

 def formats
   fetch_header("action_dispatch.request.formats") do |k|
     params_readable = begin
                         parameters[:format]
                       rescue ActionController::BadRequest
                         false
                       end

     v = if params_readable
       Array(Mime[parameters[:format]])
     elsif use_accept_header && valid_accept_header
       accepts
     elsif extension_format = format_from_path_extension
       [extension_format]
     elsif xhr?
       [Mime[:js]]
     else
       [Mime[:html]]
     end
     set_header k, v
   end
 end
 
 # 这里一旦匹配到  ,*/* | */*, 就会忽视掉整个 Accept 了。
 BROWSER_LIKE_ACCEPTS = /,\s*\*\/\*|\*\/\*\s*,/

def valid_accept_header # :doc:
    (xhr? && (accept.present? || content_mime_type)) ||
       (accept.present? && accept !~ BROWSER_LIKE_ACCEPTS)
end

```

