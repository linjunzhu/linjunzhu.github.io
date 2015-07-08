---
layout: post
title: "Rails-CSRF-Explore"
date: 2015-04-20 22:52:59 +0800
comments: true
categories: 
---

##目录

1. 引言
2. session
3. CSRF
4. 真相大白
5. 破解？

## 一、引言
这段时间一直在想，Rails 的 CSRF token 要破解不是很简单么？`GET`请求到网页拿到 token，随后发送恶意`POST`请求到服务器不就破解了么？事实证明我还是 Too young too simple.

## 二、尝试
1. 写了个程序在`ruby`中去`GET`其他服务器的网页，获取到`token`后，伪造恶意`POST`请求，附带`token`，结果`402 invalid token`


2. 真实打开浏览器，获取网页中`token`，随后复制该`token`，在`ruby`中恶意伪造`POST`请求并附带`token`，结果还是`402 invalid token`

3. 真实打开浏览器，获取token，打开`postman`，发送`POST`请求。成功`200`

这是为什么呢？

## 二、session
在 Rails 中，默认的session存储方式是：`ActionDispatch::Session::CookieStore`

也就是默认将`session`的内容存放到客户端`cookie`中。

```ruby
_glassx_session:
TE9xZ3Zud1dxRFYxdEtSek5mYldTMkpnZ1NWUlF5SjdzODVRSkJYNGN6VmNDc1VGb1lzSGJPU0FLYWhoMU5ZSHZCeXUwNTFWdWFQaWpKZmZSUC96c0dWbFdDcmlZK3RTcENabXZoaFVScWx1SWlxR1dEbmcwU1BXSDBZWUVOVW1EN0ZscmZpWkJsOFBZajZST0Z3VWxJM09NOGVTRFp2djRyYVB4SVJZNkVWRlM4dmw0TVNYb01jOGJYdVRXKzYxY1pyeXNtT0VkVWZ4YjZFcTdVU1FLVEU2aXlXbGRIOWY3c0Q1THlPRGtWeFhOb1BCd1M0S3hOQ04xTHNSajh3MC0tS3I0UVZkK2tuWGx0d1BDSVp3b1diZz09--cd6094c15ae50026d0377b6c32b7e0986b447d74
```

session中的数据保存在cookie中时是先marshal后，然后利用密码来加密的。

```ruby
marshal(data)---digest_with_secret_key(marshal(data))
```

这里所用`secret key`则是我们在`config/secrets.yml`设置的。

加密的原因是防止 session 被偷看，并且`防止 session 被修改`。

## 三、CSRF
我们在访问服务器的网页时，会找到这样的 token
```ruby
<meta name="csrf-param" content="authenticity_token">
<meta name="csrf-token" content="yT5/q+GxMDy96ISmdNfE4esNrc8YZOBUrQefXO21tJ19iGD1XjRkJ2/ELC1A952U5qDp3vpo6MhhHQB8fOmivw==">
```

这`token`是服务器生成后返回的。

具体流程是：
```ruby

# 生成加密的token（也就是我们在网页上见到的csrf-token)
def masked_authenticity_token(session)
  one_time_pad = SecureRandom.random_bytes(AUTHENTICITY_TOKEN_LENGTH)
  encrypted_csrf_token = xor_byte_strings(one_time_pad, real_csrf_token(session))
  masked_token = one_time_pad + encrypted_csrf_token
  Base64.strict_encode64(masked_token)
end

# 生成 _csrf_token，并且将其放入`session[:_csrf_token]`中
def real_csrf_token(session)
  session[:_csrf_token] ||= SecureRandom.base64(AUTHENTICITY_TOKEN_LENGTH)
  Base64.strict_decode64(session[:_csrf_token])
end

# 验证 token 是否有效（这段代码我简化过）
def valid_authenticity_token?(session, encoded_masked_token)
	masked_token = Base64.strict_decode64(encoded_masked_token)
	if masked_token.length == AUTHENTICITY_TOKEN_LENGTH
       compare_with_real_token masked_token, session
	elsif masked_token.length == AUTHENTICITY_TOKEN_LENGTH * 2
	   one_time_pad = masked_token[0...AUTHENTICITY_TOKEN_LENGTH]
       encrypted_csrf_token = masked_token[AUTHENTICITY_TOKEN_LENGTH..-1]
       csrf_token = xor_byte_strings(one_time_pad, encrypted_csrf_token)

       compare_with_real_token csrf_token, session
    end
end

# 开始验证
def compare_with_real_token(unmasked_token, session)
   ActiveSupport::SecurityUtils.secure_compare(unmasked_token, real_csrf_token(session))
end
```

从源码可以看出，服务器会将生成的`_csrf_token`放入用户的`session`中，当客户端将`csrf_token`随着`request`发送过来时，服务器会将`csrf_token`转换回非加密，随后与`session`中的`_csrf_token`进行验证。

## 四、真相大白
1. 验证时需要`session内的_csrf_token`，而用`ruby`代码直接发送http请求时是不会附带`cookies`的，所以验证肯定不会通过，这也是上面尝试第一二步失败的原因。
2. 使用`postman`是会自带`cookies`发送过去的，因此验证通过。

### Tips

附带`cookies`是浏览器的行为

## 五、破解？
这时有人会问，`rails`是开源的，有人看了源码之后，拿到网页的`csrf_token`，按照步骤一步步转换回非加密状态，不就破解了？

还是 too young!

在服务器是将`session[_csrf_token]`跟用户的`csrf_token`进行验证，就算你将`csrf_token`还原了也没用，还是需要`session[_csrf_token]`，而`session`是经过`secret`加密过的，因为无法篡改`session`，因而也破解不了。