---
layout: post
title: "Capistrano3+Puma+Nginx部署应用"
date: 2015-06-16 13:28:45 +0800
comments: true
categories: 
---

## 前言

Puma 是支持 Ruby 的一个 Web 应用服务器，基于多线程。

## 使用环境
1. Capistrano 3.x
2. Rails4.2
3. Ruby2.2
4. RVM
5. Puma
6. Nginx

## 安装
在`gemfile`中添加支持的 gem
```ruby
group :development do
  gem 'capistrano'
  gem 'capistrano-bundler'
  gem 'capistrano-rails'
  gem 'capistrano-rvm'
end

group :production do
  # puma 是一个 Gem,服务器不需要安装其他的
  gem 'puma
end
```

## 初始化
```ruby
$ cap install
```

## 配置
`capfile`
```ruby
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/rvm'
require 'capistrano/bundler'
require 'capistrano/puma'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'

Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

`deploy.rb`
```ruby
# config valid only for current version of Capistrano
lock '3.4.0'

set :application, 'xgroup'
set :repo_url, 'git@github.com:linjunzhugg/xx.git'

set :deploy_to, '/mnt/xgroup'

set :scm, :git

# Default value for :linked_files is []
set :linked_files, %w{config/database.yml config/oauth2.yml config/redis.yml config/secrets.yml}

# Default value for linked_dirs is []
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle}

set :keep_releases, 5

namespace :deploy do

  after :restart, :clear_cache do
    on roles(:web), in: :groups, limit: 3, wait: 10 do
      # Here we can do anything such as:
      # within release_path do
      #   execute :rake, 'cache:clear'
      # end
    end
  end

  after :finishing, 'deploy:cleanup'

end
```

`production.rb`
```ruby
# Simple Role Syntax

set :rvm_type, :user
set :rvm_ruby_string, 'ruby-2.2.1'

set :branch, "master"

server '0.0.0.0', user: 'deployer', roles: %w{app}

# 针对 puma 的配置
set :puma_rackup, -> { File.join(current_path, 'config.ru') }
set :puma_state, "#{shared_path}/tmp/pids/puma.state"
set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"    #accept array for multi-bind
set :puma_conf, "#{shared_path}/puma.rb"
set :puma_access_log, "#{shared_path}/log/puma_error.log"
set :puma_error_log, "#{shared_path}/log/puma_access.log"
set :puma_role, :app
set :puma_env, fetch(:rack_env, fetch(:rails_env, 'production'))
set :puma_threads, [0, 120]
set :puma_workers, 0
set :puma_worker_timeout, nil
set :puma_init_active_record, false
set :puma_preload_app, true

# 最终会创建 shared/puma.rb，将配置写进去，然后调用
```

`envrioments/production.rb`
```ruby
# Disable serving static files from the `/public` folder by default sinc
# Apache or NGINX already handles this.
config.serve_static_files = ENV['RAILS_SERVE_STATIC_FILES'].present

# important!!!
# 在 production.rb 有这么个配置，决定服务器是否伺服静态文件，默认服务器没设这个变量，因此为 false, 默认不伺服静态文件
```

## 服务器配置

`xgroup.conf`
```ruby
upstream xgroup {
   server unix:/mnt/xgroup/shared/tmp/sockets/puma.sock;
}

server {
  listen 80;
  server_name xx
  root /mnt/xgroup/current/public
  
  # 由于服务器不伺服静态文件，所以自己需要手动配置
  location ~* /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }
  
  location / {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://xgroup;
  }

}
```
```ruby
# 也可以这么做：

upstream xgroup {
   server unix:/mnt/xgroup/shared/tmp/sockets/puma.sock;
}

server {
  listen 80;
  server_name xx
  root /mnt/xgroup/current/public
  
  # 先去访问机器上对应的路径文件，访问不了再去 @xgroup
  # 如果不加这句，会访问不了静态文件  
  try_files $uri/index.html $uri @xgroup;
  location @xgroup {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://xgroup;
  }

}

```


Over~