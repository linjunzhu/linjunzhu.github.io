---
layout: post
title: "Ubuntu下修改Mysql5.5中文编码"
date: 2014-10-07 13:58:12 +0800
comments: true
categories: 
---
##前言
由于之前在 Ubuntu 下安装 Mysql时，忘记选择编码，于是全部都是 Latin 编码，折腾了一阵子搞定了

## Begin
在网上搜了一阵教程，明白 Mysql 安装后的一系列目录：

```ruby
/etc/init.d/mysql  ( 启动脚本）
/etc/mysql/my.cnf ( MySQL 的配置文件 )
/var/lib/mysql   ( 数据库文件存放路径）
/usr/lib/mysql  （Mysql 安装路径

命令：service mysql start / stop / restart
```

Mysql 版本： 5.5以上

##Doing
按照网上教程，打开 `my.cnf`，在[client] [mysqld] 下加上
`default-character-set=utf8`

结果启动mysql报错，于是我查看了`/var/log/mysql/error.log`的日志，这里用`cat`命令load不出东西，要直接vim进去才行，奇怪。

报错原因：`unknown variable 'default-set-server=utf8'`

最终解决：
Mysql 版本5.5 以上，需要这么做：

```ruby
在[client]下添加：

default-character-set=utf8
 
在[mysqld] 下添加：
 
character-set-server=utf8

在[mysql] 下添加

default-character-set=utf8
```

然后重启mysql

`sudo service mysql restart`

进入mysql终端
```ruby
show variables like '%character%';
#会有如下显示：
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       |/usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
```

