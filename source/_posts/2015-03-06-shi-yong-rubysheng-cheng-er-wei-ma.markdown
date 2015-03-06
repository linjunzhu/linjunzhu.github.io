---
layout: post
title: "'使用Ruby生成二维码'"
date: 2015-03-06 15:53:15 +0800
comments: true
categories: 
---
##前言
偶尔项目中会用到二维码，这时生成二维码有几种方式：

1. 自己动手，丰衣足食，自己来生成
2. 通过调用网络API来拉取二维码

但是，由于担心网络的不稳定 或者 项目只能对内部开放，此时就需要第一种方式来实现了。

##所需 Gem

1. gem 'qrencoder'
2. gem 'rqrencoder-magick'
3. gem 'rqrcode_png'

## Begin
###1. 先安装本机库 `qrencode`

Mac 用户：
```
brew install qrencode
```

Linux 用户：
```
apt-get install qrencode
```

###2. 安装 `qrencoder` 这个 gem

项目主页： [https://github.com/harrisj/qrencoder](https://github.com/harrisj/qrencoder)

安装之前需要先安装依赖库，否则会安装不上：
```bash
apt-get install libqrencode-dev python-qrencode qrencode
```
安装时需要指定路径：
```bash
gem install qrencoder –with-opt-include=/usr/local/include –with-opt-lib=/usr/local/lib
```
`gem qrencoder`

###3. 安装 `rqrencoder-magick` 和 `rqrencoder`
前者是利用了`RMagick`来生成二维码，因此需要事先安装 `RMagick`，安装方法就不说了。

在 `gemfile`
```
gem 'rqrencoder-magick'
```

在`shell`
```
gem install rqrencoder –with-opt-include=/usr/local/include –with-opt-lib=/usr/local/lib
```

###4. 安装`rqrcode_png`
`gem rqrcode_png`

这个 Gem 可以为生成的二维码指定`宽度`和`高度`

###5. `DEMO`
```
# 生成二维码
qr = RQRCode::QRCode.new( self.code, :size => 1, :level => :l )
png = qr.to_img  
png.resize(300, 300).save(path)
```


###6. 不足

这种方式生成的二维码图片会比现今网上提供的api所生成的二维码要「难扫」「复杂」，所以如果对二维码的扫描难度有要求的话，就....