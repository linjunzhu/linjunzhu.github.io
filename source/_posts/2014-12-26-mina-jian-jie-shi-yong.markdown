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
```ruby
# 当 rake 退出时会调用。
  def mina_cleanup
    run! if commands.any?
  end
```
6、`command` 包含所有已经 queue 的任务
```ruby
queue "sudo restart"
queue "true"

to :clean do     # 这里 clean 的queue 会被放到 :clean 命名空间下
  queue "rm"
end

commands == ["sudo restart", "true"]
commands(:clean) == ["rm"]


注意了，commands 是一个方法，而不是变量

# 源代码
def commands(aspect=:default)
  (@commands ||= begin
    @to = :default
    Hash.new { |h, k| h[k] = Array.new }
  end)[aspect]
end

因此如果要清空 commands，应该是执行:

@commands[:default] = []
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

8、已被 invoke 的 task, 再次被 invoke 时，是不会再次被执行的，除非：
```ruby
# 该参数表明再次invoke时会继续执行一次。
invoke :setup, {reenable: true} 

# 注意，任务是否已经被 invoke 过，并不是通过 commands 内的值来判断的，因此就算 commands 清空了，task 还是会被标记成已 invoke 过的。
```

9、
```ruby
task :hello do
  set :deploy_to, 'hellooooooooo'
end

task :world do
  p deploy_to
end

$-> mina hello world 

===>  'hellooooooooo'

说明 mina task1 task2 是可以串行执行的，并且任务1的环境会跟任务2相连。（同个执行环境）
通常用在设置不同的 stage
```

##三、多机部署


### 一、将同一项目部署到多个服务器
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
set :domains, %w[192.168.0.12 192.168.0.13 192.168.0.14]
```
那么效果会是：
```ruby
 begin to deploy server 192.168.0.12

    deploying.....

 finish to deploy server 192.168.0.12

 begin to deploy server 192.168.0.13

    deploying.....

 finish to deploy server 192.168.0.13
 
 begin to deploy server 192.168.0.14

    deploying.....

 finish to deploy server 192.168.0.14
 
    deploying......  ( server 192.168.0.14 )
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
###原因
在 rake 退出前会自动调用 `run!` 这个方法，因此到最后是会多执行一次 commands 内的命令的。
```ruby
def mina_cleanup!
  run! if commands.any?
end
```

###解决方法

```ruby

set :domains, %w[192.168.0.12 192.168.0.13]


desc "multi deploy"
task :multi_deploy do
   isolate do   # =>  开辟一个新的 block (这样isolate外面的commands为[]）
    domains.each_with_index do |domain, index|
      p "begin to deploy#{domain}"
      set :domain, domain
      # 这里其实只 invoke 了一次，之所以执行了三次，是因为有三次 run!
      # 而 domain 参数并不是写进 commands 的，而是在 run! 的时候进行处理
      # 因此这里可以成功的分别部署到三台机器
      invoke :deploy
      run!
      p "finish to deploy"
    end
  end
  # rake 任务退出时执行 run!, 但此时 commands 为 []
end
```


###二、将同一项目部署到同一机器的两个目录
因为该项目需要做全站国际化，因此我将它分别部署为两套 server.

```ruby
task :deploy_all do
  isolate do
    2.times do |sequence|
      p "begin to deploy #{sequence}"
      if sequence == 0
        set :deploy_to, '/home/deploy/xshare_en'
      else
        set :deploy_to, '/home/deploy/xshare_zh'
      end
      invoke :deploy
      run!
      p "end to deploy #{sequence}"
    end
  end
end
```

这么做的后果是：部署了两次到 `/home/deploy/xshare_en`去了。

###原因
1、 `deploy_to` 的值是直接写进 `commands` 内的
```ruby
[...Setting up /home/deploy/xshare_zh\" && (\n  mkdir -p \"/home/deploy/xshare_zh\" &&\n  chown -R `whoami` \"/home/deploy/xshare_zh\" &&\n  chmod g+rx,u+rwx \"/home/deploy/xshare_zh\" &&\....]
```

2、这里 `:deploy` 只 `invoke` 了一次，因此 commands 内的值是没有变化的，所以相当于执行了两次一样的第一次`commands`

###尝试1
```ruby
task :deploy_all do
  isolate do
    2.times do |sequence|
      p "begin to deploy #{sequence}"
      if sequence == 0
        set :deploy_to, '/home/deploy/xshare_en'
      else
        set :deploy_to, '/home/deploy/xshare_zh'
      end
      # 重复 invoke
      invoke :deploy, reenable: true
      run!
      # 清空 commands
      @commands[:default] = []
      p "end to deploy #{sequence}"
    end
  end
end
```
结果还是没达到预料中的结果。

因为`task deploy` 中还 `invoke` 了其他的 `task`,比如`git:clone`，因此其他的`task`只算invoke了一次，就算我手动将`deploy`所引用的`task`全部都加`reenable`参数，还是不行，因为难保其他task没invoke另外的task

###尝试2
```ruby
task :deploy_all do
  2.times do |sequence|
  # 尝试将两者区分开block
    isolate do
      p "begin to deploy #{sequence}"
      if sequence == 0
        set :deploy_to, '/home/deploy/xshare_en'
      else
        set :deploy_to, '/home/deploy/xshare_zh'
      end
      # 重复 invoke
      invoke :deploy
      run!
      # 清空 commands
      @commands[:default] = []
      p "end to deploy #{sequence}"
    end
  end
end
```

但还是失败，:deploy 是否被 invoke,跟 isolate 无关，因为 isolate 控制的只是 commands 而已。

###解决

`没找到好办法，暂时分开task执行，不过这种使用场景也比较少。`


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

这是个难点。我还没找到解决方法。只能说分开来执行,或者写脚本根据不同服务器执行不同命令