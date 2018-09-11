---
layout: post
title: "OVER THE WALL"
date: 2016-02-17 15:31:30 +0800
comments: true
categories: 
---
#OVER THE WALL
```
小时候，自由是一张破旧的五角钱，我在这头，美食在外头。

长大了，自由是一纸灼人的高考卷，我在这头，大学在外头。

后来啊，自由是一道宏伟的柏林墙，我在这头，世界在外头。
```

"子轩，把那个什么门发我下。"


犹记得第一次接触「wall」的概念，还是大学第二学期，在听子轩介绍完 "自由门" 的神奇之后，就立马让他发给了我，从此打开了 "新世界" 的大门~

## 工具

现在的翻墙方式挺多的，主流的有：

1. VPN
2. 代理
3. ShadowSocks

## VPN

VPN 相当于虚拟出一个虚拟网卡，流量全部走这个虚拟网卡，而如果做了智能分流(修改路由表)的话，访问国内网站会走真实网卡，其他的走虚拟网卡。

VPN 推荐：[云梯](http://igotvpn.com/?r=cc22a7c3d700276e)

## 代理

顾名思义，将要访问的网站将会通过「代理服务器」来进行中转，操作简单，只需设置下代理地址即可。

代理 推荐：[轻云](http://theqingyun.info/r/dkp2pm)

## ShadowSocks
SS 则是通过 SOCKS5 代理的方式，如果你是用 Mac 的话，点开 Wifi 设置 => 高级，就会看到代理的选项。

能否代理要看使用的软件支不支持 SOCKS5 代理,有些软件只支持 HTTP 代理，那就没辙了

```
Chrome : 点开设置你会发现它是默认是用系统代理设置的
QQ : 不使用系统代理设置，可以自定义代理设置
邮件: 默认系统代理设置
```

SS 推荐: [crola.xyz](crola.xyz) ( 现在注册需要邀请码，有需要可以找我拿 )

## SHELL 无法代理？
由于 VPN 是全局流量，因此 SHELL 的流量也是走 VPN 的，因此也处于「翻墙状态」

但 代理 & ShadowSocks 的方式不一样，需要程序支持

而SHELL 只支持 HTTP 代理，并且需要如下设置:

```ruby
# .zshrc / .bashrc 等配置文件之一
export http_proxy=proxy_addr:port
```

如此 SHELL 也处于了「翻墙」的状态了。

但是 ShadowSocks 是通过 SOCKS5 的方式，并不支持 HTTP 流量，因此我们需要工具来将流量转换成 SOCKS5 并且转发给 SS 代理服务。

常见的工具有 ProxyChains-NG、polipo

## ProxyChains-NG

项目主页： [https://github.com/rofl0r/proxychains-ng](https://github.com/rofl0r/proxychains-ng)

### 安装
```ruby
brew install proxychains-ng
```

### 配置

```
$ vim /usr/local/etc/proxychains.conf
```

```ruby

# edit
strict_chain
proxy_dns 
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
quiet_mode

[ProxyList]
socks5  127.0.0.1 1080

# 本地开启的服务正是 1080 端口
```

### 使用
```ruby
proxychains4 -q curl 'google.com'

## 添加 alias(在 .zshrc / .bashrc ....)
alias ss="proxychains4 -q"

## 简化
ss curl 'google.com'
```

### 过程
```ruby
SHELL 发出 request => ProxyChains-NG 将 HTTP 流量转换成 SOCKS5 流量 =>

本地 SS 服务 转发请求到 远程 SS 服务器 => SS 服务器请求实际 request, 将 response 返回
```
### 10.11
由于 Mac OSX 10.11 增加了安全机制，导致无法使用，唯一方法就是关闭 SIP

1. 首先重启系统，在开机时按住 Command+R 来进入 Recovery 模式
2. 菜单栏中找到 terminal, 打开
3. 执行: csrutil disable
4. 如果想恢复，可以执行: csrutil enable



## polipo

```
brew install polipo
```

```
vim /usr/local/opt/polipo/homebrew.mxcl.polipo.plist
```

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
 <plist version="1.0">
   <dict>
     <key>Label</key>
     <string>homebrew.mxcl.polipo</string>
     <key>RunAtLoad</key>
     <true/>
     <key>KeepAlive</key>
     <true/>
     <key>ProgramArguments</key>
     <array>
       <string>/usr/local/opt/polipo/bin/polipo</string>
       <string>socksParentProxy=localhost:1080</string>
     </array>
     <!-- Set `ulimit -n 65536`. The default macOS limit is 256, that's
          not enough for Polipo (displays 'too many files open' errors).
          It seems like you have no reason to lower this limit
          (and unlikely will want to raise it). -->
     <key>SoftResourceLimits</key>
     <dict>
       <key>NumberOfFiles</key>
       <integer>65536</integer>
     </dict>
   </dict>
 </plist>
``` 
 
开机自启动

```
ln -sfv /usr/local/opt/polipo/*.plist ~/Library/LaunchAgents
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.polipo.plist
```

```
vim ~/.zshrc
```

```
alias proxy="export http_proxy=http://localhost:8123;export https_proxy=http://localhost:8123"  # 开启
alias unproxy="unset http_proxy"  # 关闭
```

```
# 开始使用代理
$ proxy
$ todo ...


# 取消使用代理
$ unproxy
$ todo ...
```

## Ping 命令

```ruby
# 有人会发现 ping 无法翻墙
ss ping google.com
```
因为 Ping 命令是使用 ICMP 协议，而 ProxyChains 只支持 TCP and DNS, So ~~

[ http://proxychains.sourceforge.net/ ]( http://proxychains.sourceforge.net/ )