---
layout: post
title: "Rails中的缓存机制"
date: 2014-10-05 11:43:10 +0800
comments: true
categories: 
---
##1、前言
之前一直对缓存处于模模糊糊的概念，昨天下定决心好好恶补一下，因此写下這篇文章做下总结，只有简单的知识点，木有高深内容:  )

##2、前端缓存
我们平常所使用的浏览器，是有带缓存区的。

![](http://data-storage.qiniudn.com/%E5%89%8D%E7%AB%AF%E7%BC%93%E5%AD%98%E5%9B%BE.png)

图不太好画啊，泪流满面。

图中的“文字太长，下面解释”：

```
浏览器会附带`If-None-Match`属性（内容id ) 或 `If-Modified-Since`（修改内容时间戳）

服务器的响应头会附带：

1. ETag ： 对应请求的 `If-None-Match`属性
2. Last-modified：对应请求的`If-modified-Since`属性

两者有一个相同就返回 `304`
```

###2.1、静态文件请求
当使用 WEB 服务器时，如： Nginx，是会自动处理静态文件的缓存的，而不会通过服务器。当然，也是根据 `ETag` 和 `Last-modified` 来判断

###2.2、动态文件请求(这里其实不算前端范畴了）
当请求动态文件时，就需要通过服务器来处理了，在 Rails 中默认使用`Rack::ETag middleware`中间件，它会自动给无 ETag 的 response 加上 ETag，再根据 ETag 来判断资源是否更改，从而返回 200 / 304。

`注意`: 返回304并不是说服务器没有处理资源直接就返回304，而是完成的处理完了资源，生成了response准备响应了，才进行生成 ETag，然后进行判断，这样虽然也会消耗服务器的资源，但是可以大大减少客户端传输网络的时间。

之前有个疑惑：
```
为什么要等到生成完整 response 再加上 ETag 进行判断呢？我的想法是：因为判断
资源是否有变化，所以需要判断`response内的某些资源`是否有变化，因此迟早都要生成response，
所以是否等到生成完整response再加上ETag是无所谓的
```


补充几点 :

1. 发送「条件GET请求」是需要服务器的支持的。
2. HTTP在传输的时候是先给状态再给内容


###2.3、条件GET请求简介
如果浏览器缓存中已经有一个 HTML 内容元素的副本，却不能保证该缓存是否有效，这时，一个条件 GET 请求就出现了。

通常情况下，缓存是否有效取决于最后一次修改时间，而浏览器是根据返回报头中的 Last-Mofidied 属性来获取到这个时间，浏览器则会在请求报头中加入 If-Modified-Since 属性到服务端。 
![](http://data-storage.qiniudn.com/%E6%9D%A1%E4%BB%B6get%E8%AF%B7%E6%B1%82.png)
如果时间点一致，则直接303，跳过响应的内容体。

##3、Rails的缓存机制

服务器端缓存区存储有两种方式：

1. 硬盘
2. 内存

运行流程图：

![](http://data-storage.qiniudn.com/%E5%90%8E%E7%AB%AF%E7%BC%93%E5%AD%98.png)

画得有点呵呵，将就着看吧。

说明几点：

1. 每次请求都会重新计算`cache_key`，而计算`cache_key`则会进入到数据库去查询，比如`products`需要查询`products.max(&:updated_at)`。
2. 缓存本质倒不是查不查，而是把一个长查询转化成短查询。
3. 当资源更新时，对应缓存区的缓存会过期，但过期不是删除，缓存还是会存在缓存区中，只是不再命中，`所以基于 file 的缓存方案会产生许多重复的垃圾，dhh 推荐使用 memcache 就是把删除缓存的操作交给 memcache 来做。`


##4、Rails中使用缓存
在 Rails 中使用缓存是个大学问

###4.1、fresh_when
前文说到，Rails 默认的中间件`Rack::ETag middleware`会自动给无`ETag`的`response`加上`ETag`，那么，有没办法在生成完整的response之前（节省资源）我们自己加上`ETag`呢？ Rails 提供了 `fresh_when` helper

```ruby
class ArticlesController
  def show
    @article = Article.find(params[:id])
    fresh_when :last_modified => @article.updated_at.utc, :etag => @article
  end
end
```
下次用户再访问的时候，会对比request header里面的If-Modified-Since和If-None-Match，如果相符合，就直接返回304，而不再生成response body。

但是这样会遇到一个问题，假设我们的网站导航有用户信息，一个用户在未登陆专题访问了一下，然后登陆以后再访问，会发现页面上显示的还是未登陆状态。或者在app访问一篇文章，做了一下收藏，下次再进入这篇文章，还是显示未收藏状态。解决这个问题的方法很简单，将用户相关的变量也加入到etag的计算里面：
```ruby
 fresh_when :etag => [@article.cache_key, current_user.id]
 fresh_when :etag => [@article.cache_key, current_user_favorited]
```

`和fresh_when相比，自动etag能够节省的只是客户端时间`

###4.2  Fragment Caching
Rails  中有几种 cache，这种是最常见使用最多的。

如下，有个项目页面：
![](http://data-storage.qiniudn.com/%E5%A5%97%E5%A8%83%E6%9C%BA%E5%88%B6.jpg)

接下来我们看看，如果这个页面的某一条数据，比如「任务 A-1」 的内容被改变了，会发生什么。

首先，这条任务自己对应的 L4 Todo Item 缓存失效了，所以在拼装外面的 L3 级「任务清单A」缓存的时候，会从缓存里获取任务 A-2、A-3 的缓存，速度嗖嗖快，快到可以忽略不计，然后对任务 A-1 重新渲染一次，放入缓存，这样「任务清单A」通过直接从缓存里读取两条任务（A-2 和 A-3），以及渲染一条新的（ A-1 ）生成了整个 L3 Todolist Item 的页面片段。剩下的「任务清单B」和「任务清单C」，都没有变化，因此由在生成「任务清单」Section 缓存的时候，直接拼装即可。

其它几个 Section 片段因为和任务没有任何关系，所有缓存都不会过期，因此这几个 Section 的页面片段都是直接从缓存里捞出来，同样嗖嗖快。

最后，整个项目详情页把这几个 Section 拼装起来，返回给客户。从上面的过程可以看出，只有「任务 A-1」 这个片段的页面被重新渲染了。

所以，这种套娃式的缓存，能够保证页面缓存利用率的最大化，任何数据的更新，只会导致某一个片段的缓存失效，这样在组装完整页面的时候，由于大量的页面片段都是直接从缓存里读取，所以页面生成的时间开销就很小。

那么，套娃是如何在缓存中存取页面片段的呢？主要是靠一个叫做 cache_key 的东西来决定的。

###4.2.1、cache_key
代码初级版：
```ruby
<% cache @project do  %>
  <%= @project.name %>

  <!-- Section Topics -->
  <% cache @top3_topics.max(&:updated_at) do %>
    <%= render partial: 'topics/topic', collection: @top3_topics %>
  <% end %>

  <!-- Section todolists -->
  <% cache @todolists.max(&:updated_at) do %>
    <%= render partial: 'todolists/todolist', collection: @todolists %>
  <% end %>

  <!-- Section uploads -->
  <% cache @uploads.max(&:updated_at) do %>
    <%= render partial: 'uploads/upload', collection: @uploads %>
  <% end %>

  <!-- Section documents -->
  <% cache @documents.max(&:updated_at) do %>
    <%= render partial: 'documents/document', collection: @documents %>
  <% end %>

  <!-- Section calendar_events -->
  <% cache @calendar_events.max(&:updated_at) do %>
    <%= render partial: 'calendar_events/calendar_event', collection: @calendar_events %>
  <% end %>

<% end %>
```

###单个资源
可以看到，最外层有个`<% cache @project %>`，这个`cache`使用`@project`作为参数，cache内部会对该对象进行处理，计算出`cache_key`

大概是：`views/projects/1-20140906112338`

Rails 会使用该`cache_key` 作为`key`,将包含的所有东西（包括数据和html）通通放入缓存区。每次有请求到来时，就会重新计算 cache_key，然后到缓存区进行查找，命中则返回，不命中则服务器查询资源，重新将资源放入缓存区。

当资源更新时，`cache_key` 也随之变化（注意，`cache_key` 大部分需要自己设计)，因此就不会命中缓存，缓存自然过期。

`注意：` 如果不生成`cache_key`，缓存永远都不会变！！！无论资源是否更新
`注意：` 缓存过期不代表缓存被删除，因为缓存区会存在大量垃圾，需要memcached来解决。

###多个资源
那如果是资源列表呢？如果我们直接把 `@top3_topics` 对象作为 cache 的参数 `<% cache @top3_topics %>`，得到的 `cache_key` 实际上会是这样的形式：

`views/topics/3-20140906112338/topics/2-20140906102338/topics/3-20140906092338`

显然，这样做不太好，因此我们可以取这组数据内最新一个被更新的数据的updated_at时间戳，这样会生成：

`views/20140906112338`

但这样会出现一个问题，假如此时`@top_topics`为空，那么我们此时是对 nil 进行缓存，不幸的是，所有 nil 的 `cache_key` 都一模一样，这样就会造成混乱。再加上，加入任务清单 Section 和讨论 Section 最后更新的那条数据的 updated_at 时间戳恰好一样，也会造成两个缓存片混淆的问题。

解决方法就是给每个cache都加上model名
`<% cache [:topics, @top3_topics.max(&:updated_at)] %>`

代码更新版：
```ruby
<% cache @project do  %>
  <%= @project.name %>

  <!-- Section Topics -->
  <% cache [:topics, @top3_topics.max(&:updated_at)] do %>
    <%= render partial: 'topics/topic', collection: @top3_topics %>
  <% end %>

  <!-- Section todolists -->
  <% cache [: todolists,  @todolists.max(&:updated_at)] do %>
    <%= render partial: 'todolists/todolist', collection: @todolists %>
  <% end %>

  <!-- Section uploads -->
  <% cache [: uploads,  @uploads.max(&:updated_at)] do %>
    <%= render partial: 'uploads/upload', collection: @uploads %>
  <% end %>

  <!-- Section documents -->
  <% cache [: documents,  @documents.max(&:updated_at)] do %>
    <%= render partial: 'documents/document', collection: @documents %>
  <% end %>

  <!-- Section calendar_events -->
  <% cache [: calendar_events,  @calendar_events.max(&:updated_at)] do %>
    <%= render partial: 'calendar_events/calendar_event', collection: @calendar_events %>
  <% end %>

<% end %>
```

###4.2.2 touch!机制

我们回过头来再看看套娃缓存的读取机制，访问项目详情页的时候，首先读取最外层的大套娃 <% cache @project %> ，如果这个缓存片对应的 `cache_key `在缓存里能找到，则直接取出来并且返回，如果缓存过期，则读取第二级套娃 — 几个列表 Section 缓存，这些缓存根据列表里最新一条数据的更新时间生成 cache_key，如果最新一条数据的更新时间没有变化，则缓存不过期，直接取出来供页面拼装用，如果缓存过期，则继续读取各自的第三级套娃。

等等，这里有个问题，如果我改变了一条任务的内容，也就是作废了任务 partial 自己的缓存，但是包裹任务的任务清单，以及包裹任务清单的项目都没有变化，这样当页面加载的时候，读取到的第一个大套娃 -- <% cache @project %> 都没有更新，会直接返回被缓存了的整个项目详情页，所以根本不会走到渲染更新的任务 partial 那里去。对于这个问题的解决方案，是 Rails 模型层的 touch 机制。

简单的说，我们需要让里面的子套娃在数据更新了以后，touch 一下处在外面的套娃，告诉它，嘿，我更新了，你也得更新才行。我们直接看看这个代码片段：
```ruby
class Project < ActiveRecode::Base
  #...
end

class Todolist < ActiveRecode::Base
  belongs_to :project, touch: true
end

class Todo < ActiveRecode::Base
  belongs_to :todolist, touch: true
end
```

由于 touch 的存在，当 todo 的某条任务修改时，会自动通知到 todolist, 让其修改 updated_at 属性，而 todolist 也会修改到 project，一层层保证整个缓存系统的正常运作。

##5、Rails 里使用缓存的坑
```ruby
<% cache @project %>
	<% cache [:todolists, @todolists.max(&:updated_at)] do %>
 		 <span>清单所属项目： <%= @project.name %></span>
  	<%= render partial: 'todolists/todolsit', collection: @todolists %>
<% end %>
``` 
当出现以上情况时，属于父级cache的属性出现在子级cache内，当@project的内容更改了，这时 @todolists 的 cache_key 并没有改变，也就是说这段缓存没有过期，`<%= @project.name %>` 显示的还是过去的 name

解决方法：
```ruby
<% cache @project %>
	<% cache [:todolists, @project, @todolists.max(&:updated_at)] do %>
 		 <span>清单所属项目： <%= @project.name %></span>
  	<%= render partial: 'todolists/todolsit', collection: @todolists %>
<% end %>
``` 
```ruby
<% cache @project %>
	<% cache [:todolists, @project.name, @todolists.max(&:updated_at)] do %>
 		 <span>清单所属项目： <%= @project.name %></span>
  	<%= render partial: 'todolists/todolsit', collection: @todolists %>
<% end %>
``` 
这样子当修改@project时，也会修改子级的 cache_key

还有第二个坑，写得脑仁疼，不写啦，看下面参考文

##参考

总结 web 应用中常用的各种 cache: <https://ruby-china.org/topics/19389>

Cache 在 Ruby China 里面的应用:  <https://ruby-china.org/topics/19436>

说说 Rails 的套娃缓存机制:  <https://ruby-china.org/topics/21488?page=1#replies>

官方缓存讲解：<http://guides.ruby-china.org/caching_with_rails.html>