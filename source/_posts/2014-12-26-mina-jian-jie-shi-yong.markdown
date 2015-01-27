---
layout: post
title: "Mina 简介使用"
date: 2014-12-26 15:00:12 +0800
comments: true
categories: 
---
#说下 Mina

##一、基本使用
```ruby
gem 'ruby'

mina init    =>  生成 config/deploy.rb
```
```ruby
# config/deploy.rb
set :user, 'username'  # 部署用户
set :domain, 'your.server.com'   # 域名
set :deploy_to, '/var/www/flipstack.com' # 部署路径
set :repository, 'git@github.......xxx' #git 仓库
set :branch, 'master' #分支
set :shared_paths, ['config/database.yml', 'public/system' ...] #共享文件夹

然后：

1. 
task: environment do
	invoke :'rvm:use[ruby-2.0.0@rails4.0.4]'
end

2.
# 主要为 deploy 做准备，创建各个文件夹
task :setup => :environment do
  queue! %[mkdir -p "#{deploy_to}/shared/log"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/log"]

  queue! %[mkdir -p "#{deploy_to}/shared/config"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/config"]

  queue! %[mkdir -p "#{deploy_to}/shared/public/qrcodes"]
  queue! %[mkdir -p "#{deploy_to}/shared/public/qrcodesZip"]

  queue! %[mkdir -p "#{deploy_to}/shared/public/uploads"]
  queue! %[mkdir -p "#{deploy_to}/shared/public/system"]

  queue! %[mkdir -p "#{deploy_to}/shared/coupons"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/coupons"]

  queue! %[mkdir -p "#{deploy_to}/shared/couponsZip"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/couponsZip"]

  queue! %[touch "#{deploy_to}/shared/config/database.yml"]
  queue  %[echo "-----> Be sure to edit 'shared/config/database.yml'."]
end

3.

# 编写部署任务
task :deploy => :environment do
  deploy do
    # Put things that will set up an empty directory into a fully set-up
    # instance of your project.
    invoke :'sidekiq:quiet'
    invoke :'git:clone'
    invoke :'deploy:link_shared_paths'
    invoke :'bundle:install'
    invoke :'rails:db_migrate'
    invoke :'rails:assets_precompile'

    to :launch do
      queue "if [ -d #{deploy_to}/current/tmp ]
      then
        touch #{deploy_to}/current/tmp/restart.txt
      else
        mkdir #{deploy_to}/current/tmp
        touch #{deploy_to}/current/tmp/restart.txt 
      fi"
      invoke :'sidekiq:restart'      
    end
  end
end
```	

终端执行 

```ruby
mina setup

mina deploy
```

##二、应该要注意的 `point`

1、我们可以自己写 task 任务


2、queue 命令用在执行 bash 命令，如：
```ruby
task :logs do
     queue 'echo "Contents of the log file are as follows:"'
     queue "tail -f /var/log/apache.log"
end
```

3、invoke 命令用在引用已经写好的 task 任务，如：
```ruby
task :down do
     invoke :maintenance
end 

task :maintenance do
     queue 'touch maintenance.txt'
end
```

4、Mina 已经有一些已经写好的 task 任务，如：
```ruby
invoke :'git:clone'
invoke :'bundle:install'

#我一开始以为这是执行 bash 命令，让我理不清 invoke 跟 queue 的关系
```

5、`run!` 这个命令是指 SSH 进主机，然后执行所有已经 queue 的命令。这个命令会在 Rake 退出前，自动调用。

6、`command` 包含所有已经 queue 的任务
```ruby
queue "sudo restart"
queue "true"

to :clean do     # 这里 clean 的queue 会被放到 :clean 命名空间下
  queue "rm"
end

commands == ["sudo restart", "true"]
commands(:clean) == ["rm"]
```

7、`isolate` 命令会开辟一个新的 block，包含 queue 的任务
```ruby
queue "sudo restart"
queue "true"

commands.should == ['sudo restart', 'true']

isolate do
  queue "reload"
  commands.should == ['reload']
end

commands.should == ['sudo restart', 'true']
```

8、已被 invoke 的 task, 再次被 invoke 时，是不会再次被执行的。

##三、多机部署

```ruby

set :domains, %w[192.168.0.12 192.168.0.13]


desc "multi deploy"
task :multi_deploy do
    domains.each_with_index do |domain, index|
    
      p "begin to deploy#{domain}"
      set :domain, domain
      invoke :deploy
      run!    => # 注意这里一定要加上 run! ，才会立即运行命令
      p "finish to deploy"
    end
end
```

自己写的一个 task, 这时候遇到一个难题，发现 invoke :deploy ，当第一次循环的时候正常，第二次循环的时候，会部署两次。

效果是这样的：
```ruby
 begin to deploy server 192.168.0.12

    deploying.....

 finish to deploy server 192.168.0.12

 begin to deploy server 192.168.0.13

    deploying.....

 finish to deploy server 192.168.0.13

    deploying......  ( server 192.168.0.13 )
```

如果我的 domains  是
```ruby
set :domains, %w[192.168.0.12 192.168.0.13]
```
那么效果会是：
```ruby
 begin to deploy server 192.168.0.12

    deploying.....

 finish to deploy server 192.168.0.12

 begin to deploy server 192.168.0.13

    deploying.....

 finish to deploy server 192.168.0.13
 
 begin to deploy server 192.168.0.13

    deploying.....

 finish to deploy server 192.168.0.13
 
    deploying......  ( server 192.168.0.13 )
```
发觉最终总是会多部署最后一个server

于是我用
```ruby
p "命令集合：#{commands(:default)}"
```
来查看，也没有什么异常。

然后我又做了下实验：
```ruby
task :deploy_primary do
  p "begin to deploy primary #{primary_domain}"
  set :domain, primary_domain
  invoke :deploy
  invoke :mysql_sync
  run!
  p "finish to deploy primary #{primary_domain}"  
  p commands
end
```

发觉这样子也会重复部署两次，看来这是跟` invoke :deploy `这个特殊 task 有关了。

###解决方法

```ruby

set :domains, %w[192.168.0.12 192.168.0.13]


desc "multi deploy"
task :multi_deploy do
   isolate do   # =>  开辟一个新的 block ( 不过为何能解决我还没弄懂）
    domains.each_with_index do |domain, index|
      p "begin to deploy#{domain}"
      set :domain, domain
      invoke :deploy
      run!    => # 注意这里一定要加上 run! ，才会立即运行命令
      p "finish to deploy"
    end
  end
end
```

##总结点
第一次 invoke :deploy 的时候， mina 会去处理很多东西，第二次的时候就不会去处理了。因此如果这样子做：
```ruby
task :deploy_primary do
  isolate do
    set :domain, primary_domain
    invoke :deploy
  end
  
  isolate do
    set :domain, other_domain
    invoke :deploy
   end
end
```
第二个 :deploy 是不会生效的，会出毛病。因此如果要连续 invoke :deploy 的话，必须要放在同一个 isolate 下才行。

##单独执行命令

```ruby
task :mysql_sync do
  queue %[cd #{deploy_to!}/current && whenever -i 'mysql_sync']
end
```

当初单独执行这个 task  的时候，一直提示找不到 whenever 这个命令，令人纳闷，最终发现需要加上环境。

```ruby
task :mysql_sync => :environment do
  queue %[cd #{deploy_to!}/current && whenever -i 'mysql_sync']
end
```

## 重复的 queue
```ruby

set :domains, %w[192.168.0.12 192.168.0.13]


desc "multi deploy"
task :multi_deploy do
   isolate do   # =>  开辟一个新的 block ( 不过为何能解决我还没弄懂）
    domains.each_with_index do |domain, index|
      p "begin to deploy#{domain}"
      set :domain, domain
      invoke :deploy
      if index == 1
         queue 'do something one'
      else
         queue 'do something two'
      end
      run!    => # 注意这里一定要加上 run! ，才会立即运行命令
      p "finish to deploy"
    end
  end
end
```

这样子的话，当第二次循环时，会执行 `one` 和 `two` 两个 queue, 因为第一次循环时， `one` 已经 queued 了，进入了 commands，因此第二次循环时，会执行这个命令 （  执行所有 commands)

这是个难点。我还没找到解决方法。只能说分开来执行