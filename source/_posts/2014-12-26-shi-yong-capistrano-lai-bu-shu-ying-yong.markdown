---
layout: post
title: "使用 Capistrano 来部署应用"
date: 2014-12-26 14:58:57 +0800
comments: true
categories: 
---
# 使用 Capistrano 部署应用

## 前言
由于 Mina 一直以简单好用为著称，因此一直都是用 Mina 来部署应用。但是  Mina 也有局限性，之前的多机部署就遇到了难题，于是学习下如何用 Capistrano 来部署 Rails 应用

## 使用环境
1. Capistrano 3.x
2. Rails4.1
3. Ruby2.0
4. RVM

## 安装
在`gemfile`中添加支持的 gem
```ruby
group :development do
  gem 'capistrano'
  gem 'capistrano-bundler'
  gem 'capistrano-rails'
  gem 'capistrano-rvm'
end
```

## 初始化
```ruby
$ cap install
```
其中有几个比较重要的文件：

1. Capfile用来配置Capistrano
2. deploy.rb是一些共用task的定义
3. production.rb / staging.rb用来定义具体的stage的tasks。

通过 `cap -vT` 可以查看当前可用的 task

## 配置
`capfile`
```ruby
# Load DSL and Setup Up Stages
require 'capistrano/setup'

# Includes default deployment tasks
require 'capistrano/deploy'

# 配置支持的插件
require 'capistrano/rvm'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'

# Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

`deploy.rb`
```ruby
# config valid only for Capistrano 3.1
lock '3.2.1'

set :application, 'student_lottery'
set :repo_url, 'git@bitbucket.org:linjunzhugg/student_lottery.git'

# Default branch is :master
# ask :branch, proc { `git rev-parse --abbrev-ref HEAD`.chomp }.call

# Default deploy_to directory is /var/www/my_app
set :deploy_to, '/home/deploy/student_lottery'

# Default value for :scm is :git
set :scm, :git

# Default value for :format is :pretty
# set :format, :pretty

# Default value for :log_level is :debug
# set :log_level, :debug

# Default value for :pty is false
# set :pty, true

# Default value for :linked_files is []
set :linked_files, %w{config/database.yml}

# Default value for linked_dirs is []
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}

set :rvm_ruby_version, 'ruby-2.0.0-p481@rails4.0.4'

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for keep_releases is 5
# set :keep_releases, 5

namespace :deploy do

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      # Your restart mechanism here, for example:
      # execute :touch, release_path.join('tmp/restart.txt')
    end
  end

  # 就是这里插入的任务
  after :publishing, :restart

  after :restart, :clear_cache do
    on roles(:web), in: :groups, limit: 3, wait: 10 do
      # Here we can do anything such as:
      # within release_path do
      #   execute :rake, 'cache:clear'
      # end
    end
  end

end

```

`config/production.rb`
```ruby
set :branch, 'master'
server '192.168.0.13', user: 'deploy', roles: %w{web}
set :rails_env, :production
```

## 开始部署
```ruby
$ cap production deploy  # 开始部署
```

这时有人疑惑了，咦？这条命令的 task 在哪呢？我要如何定义自己的task 呢？

根据官方文档：一旦运行 `cap production deploy` ，就默认有以下这些任务：
```ruby
deploy:starting    - start a deployment, make sure everything is ready
deploy:started     - started hook (for custom tasks)
deploy:updating    - update server(s) with a new release
deploy:updated     - updated hook
deploy:publishing  - publish the new release
deploy:published   - published hook
deploy:finishing   - finish the deployment, clean up everything
deploy:finished    - finished hook
```

而我们之前在 `capfile` 中添加了以下的代码：
```ruby
# 配置支持的插件
require 'capistrano/rvm'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
```

这些会自动在 starting  updating 等任务中插入 before after 任务，例如：
```ruby
deploy
  deploy:starting
    [before]
      deploy:ensure_stage
      deploy:set_shared_assets
    deploy:check
  deploy:started
  deploy:updating
    git:create_release
    deploy:symlink:shared
  deploy:updated
    [before]
      deploy:bundle
    [after]
      deploy:migrate
      deploy:compile_assets
      deploy:normalize_assets
  deploy:publishing
    deploy:symlink:release
  deploy:published
  deploy:finishing
    deploy:cleanup
  deploy:finished
    deploy:log_revision
```

从中我们可以看到里面有 bundle   migrate 等任务。

## 其他插件
比如 sidekiq 的自动重启，可以加上 `capistrano-sidekiq`  gem