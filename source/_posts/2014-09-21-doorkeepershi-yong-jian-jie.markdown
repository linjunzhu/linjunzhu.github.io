---
layout: post
title: "doorkeeper使用简介"
date: 2014-09-21 21:39:25 +0800
comments: true
categories: 
---
###1、doorkeeper 是什么？
最近在项目里需要构建自己的一套 Oauth2.0，而doorkeeper正是帮助我们构建 Oauth2.0 的gem

###2、安装
```ruby
gem 'doorkeeper'
```
and then run the order

```ruby
rails g doorkeeper:install
```
It will install the doorkeeper initializer into `doorkeeper.rb`

And next step If U use `Active Record`:
```ruby
rails g doorkeeper:migration
```

###3、配置
这时你会发现在你的routes.rb里添加了这样一段话：
```ruby
Rails.application.routes.draw do
  use_doorkeeper
  # your routes
end
```
对应生成：
```ruby
GET       /oauth/authorize/:code
GET       /oauth/authorize
POST      /oauth/authorize
PUT       /oauth/authorize
DELETE    /oauth/authorize
POST      /oauth/token
POST      /oauth/revoke
resources /oauth/applications
GET       /oauth/authorized_applications
DELETE    /oauth/authorized_applications/:id
GET       /oauth/token/info
```

###4、权限
为了使用户访问 API 时，有权限，需要：
```ruby
Doorkeeper.configure do
  resource_owner_authenticator do
    User.find_by_id(session[:current_user_id]) || redirect_to(login_url)
  end
end
```

如果使用`devise`，可以添加下面这段代码
```ruby
resource_owner_authenticator do
  current_user || warden.authenticate!(:scope => :user)
end
```

###5、保护API
It needs to add the follow block for protecting API
```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action :doorkeeper_authorize! # Require access token for all actions

  # your actions
end
```

###6、访问不同权限的API

在 `doorkeeper.rb`中设置权限范围
```ruby
Doorkeeper.configure do
  default_scopes :public # if no scope was requested, this will be the default
  optional_scopes :admin, :write
end
```
要求不同action需要不同权限
```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action -> { doorkeeper_authorize! :public }, only: :index
  before_action only: [:create, :update, :destroy] do
    doorkeeper_authorize! :admin, :write
  end
end
```

###7、开始使用
我们的 Authorization Server 已经盖好了，此时我们申请个 client 来使用。
访问`/oauth/applications`来申请client

####7.1 开client
其中 Client 的 redierct URI 填入 `http://localhost:12345/auth/demo/callback` ，實際上沒有跑 Web server 在 localhost:12345 也沒關係，最終目的是拿到 code 或 token
![](http://user-image.logdown.io/user/2580/blog/2567/post/145023/1wLQZN9CS9SixjFgRaq1_oauth2-new-client.png)

####7.2 获取 access token
首先打開剛剛生的 Client 的 show 頁面，會看到有 Application ID 、 Secret 等資訊的頁面。最下面有一個 Authorize 的連結，點下去會打開到這個網址
```ruby
http://localhost:9999/oauth/authorize
    ?client_id=4a407c6a8d3c75e17a5560d0d0e4507c77b047940db6df882c86aaeac2c788d6
    &redirect_uri=http%3A%2F%2Flocalhost%3A12345%2Fauth%2Fdemo%2Fcallback
    &response_type=code
```
点击`Authorize`后
```ruby
http://localhost:12345/auth/demo/callback
    ?code=21e1c81db4e619a23d4ed46134884104225d4189baa005220bd9b358be8b591a
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          Grant Code   
```

使用`postman`来获取access_token

![](http://user-image.logdown.io/user/2580/blog/2567/post/145023/YZa3unuQgSf2kUkoSMDF_oauth2-token-request-zh.png)

### 8、撤销授权

```ruby
curl -F token=53cff8f4a549beb1c38704158b0f6608a2382f094b6947ecc35c2eed4146a17c \
 -H "Authorization: Bearer 53cff8f4a549beb1c38704158b0f6608a2382f094b6947ecc35c2eed4146a17c" \
 -X POST localhost:3000/oauth/revoke
 ```


### Tips

1. refresh_token 一旦使用就会变化。
2. 后台验证token是否有效（doorkeeper_authorize! 会自动验证是否有效。）
3. 1.x的时候 /oauth/revoke 是无效的， 2.x 才有效 （就是手动撤销对用户的授权 )