---
layout: post
title: "在 Ruby 中用 SSH 来跟服务器进行通信"
date: 2014-12-26 15:01:21 +0800
comments: true
categories: 
---
#在Ruby中用SSH跟服务器进行交流

## 前言
Net::SSH和Net::SCP是两个Ruby操作SSH的gem包。

Net::SSH相当于cmd，专门用于执行命令；

Net::SCP专门用于传输文件。

它们俩结合，可以做任何SSH client能做的事情。
##所需 Gem
1. gem 'net-ssh'
2. gem 'net-scp'

## Begin

```ruby
Net::SSH.start(MYSQL_SYNC_CONFIG["#{sequence}_server_ip"], MYSQL_SYNC_CONFIG["#{sequence}_server_user"]) do |ssh|

        # 创建文件夹
        ssh.exec!(
          "if [ ! -d #{MYSQL_SYNC_CONFIG['endpoint_backup_path']} ]; then\n" \
          "mkdir #{MYSQL_SYNC_CONFIG['endpoint_backup_path']};" \
          "fi"
        )

        # 开始上传
        ssh.scp.upload!("#{backup_sql_path}", "#{MYSQL_SYNC_CONFIG['endpoint_backup_path']}/#{backup_sql_name}")

        # 客户端开始还原
        ssh.exec!("mysql -u" + MYSQL_SYNC_CONFIG["#{sequence}_server_mysql_user"] +
                  " -p" + MYSQL_SYNC_CONFIG["#{sequence}_server_mysql_pwd"] +
                  " red_mansions < #{MYSQL_SYNC_CONFIG['endpoint_backup_path']}/#{backup_sql_name}")      
      end
```

##引用
[http://rubyer.me/blog/1133/](http://rubyer.me/blog/1133/)

[http://www.infoq.com/cn/articles/ruby-file-upload-ssh-intro](http://www.infoq.com/cn/articles/ruby-file-upload-ssh-intro)

[https://gist.github.com/lajunta/7305741](https://gist.github.com/lajunta/7305741)