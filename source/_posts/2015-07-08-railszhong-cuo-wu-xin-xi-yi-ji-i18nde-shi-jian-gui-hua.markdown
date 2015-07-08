---
layout: post
title: "Rails中「错误信息」的实践规划"
date: 2015-07-08 11:15:01 +0800
comments: true
categories: 
---
#前言
在我还是一个 Rails 新手的时候，曾纠结过，如何更好的规划显示错误信息，以及实现 I18N。前几天整理一个实习生的代码，发现他对错误信息的规划毫无章法，所以就写下这 blog, Rails 新手可以看看。

#基本概念
```ruby
class Account < ActiveRecord::Base
  validates_presence_of :email, :message => "Email 不得为空"
end

account = Account.new

# 此时会调用 account.valid?，该 valid? 会调用所有validate，通过则 true,否则为 false
account.save   => false
account.valid? => false
account.errors.empty?  => false
account.email = 'linjunzhugg@gmail.com'
account.valid? => true
account.save   => true
```

来看下源码
```ruby
# File activemodel/lib/active_model/validations.rb, line 331
def valid?(context = nil)
  current_context, self.validation_context = validation_context, context
  errors.clear
  run_validations!
ensure
  self.validation_context = current_context
end

# File activemodel/lib/active_model/validations.rb, line 394
def run_validations! 
  _run_validate_callbacks
  errors.empty?
end
```
从上面可以看到，valid? 会将验证结果放在对象的 errors中，如果对象的 errors 没东西，那么会返回 true，验证通过，返回 false，则验证不通过，save 会失败。

#开始实践
```ruby
# user.rb
class Account < ActiveRecord::Base
  # 不要在这里手动写 message，应该在 locals 中的 yml 文件写
  # validates_presence_of :email, :message => "Email 不得为空"
  validates_presence_of :email
end

# users_controller.rb
def create
  @user = User.new(user_params)
  if @user.save
    render :index
  else
    # save 失败后会将错误信息存入  @user.errors
    # 就不用自己在这里写错误信息了
    render :new
  end
end
```
Rails 有默认的错误信息，我们可以根据需要自定义错误信息

#自定义错误信息

`validates_presence_of`的错误信息 key 是 `blank`，Active Record 会按照下面的顺序寻找`blank`:

```ruby
activerecord.errors.models.user.attributes.name.blank
activerecord.errors.models.user.blank
activerecord.errors.messages.blank
errors.attributes.name.blank
errors.messages.blank
```

如果有继承链，则会往继承链上找：
```ruby
class Admin < User
  validates :name, presence: true
end
```
```ruby
activerecord.errors.models.admin.attributes.name.blank
activerecord.errors.models.admin.blank
activerecord.errors.models.user.attributes.name.blank
activerecord.errors.models.user.blank
activerecord.errors.messages.blank
errors.attributes.name.blank
errors.messages.blank
```

那么我们可以在 locales 内的 yml 文件中做文章:
```ruby
zh-CN:
  activerecord:
    attributes:
      user:
        email: 'Email'
        password: '密码'
      work_order:
        title: '标题'
        question: '问题'
    errors:
      # 执行 errors.full_message 会按照此格式输出
      format: ! '%{attribute}%{message}'
      # 这里作为通用的错误信息
      messages:
        blank: "%{attribute} 不能为空"
        invalid: "%{attribute} 格式不正确"
        taken: "%{attribute} 已经被使用"
      # 这里作为针对性的 model 错误信息
      models:
        user:
          attributes:
            password:
              too_short: '密码 不得小于6位'
```

###Tips
```ruby
zh-CN:
  activerecord:
    attributes:
      user:
        email: 'Email'
        
  avtiverecord:
    errors:
      messages:
        blank: "%{attribute} 不能为空"
        
不要这么写，这样后面的 activerecord 会覆盖掉前面的东西，而不是作为一个 namespace 的存在。
```

#显示错误信息
我们回到 controller 再看下：
```ruby
# users_controller.rb
def create
  @user = User.new(user_params)
  if @user.save
    render :index
  else
    # 直接在 view 层去显示错误信息
    render :new
  end
end
```

```ruby
# users_helper.rb
def show_error model, column
  if model.send('errors')[column][0]
    sanitize "<div class='error-msg'>* #{model.send('errors')[column][0]}</div>"
  end
end

# users/new.html.erb

<html>
<body>
  oooooooooxxxxxxxxx
  # 此时这里就可以按照需求显示出我们在 yml 中自定义的错误信息了
  show_error(@user, :avatar)
</body>
</html>
```

# 非 Active Record 操控错误信息
``` ruby
# users_controller.rb

def update
  if user_pass?
    # 自己在 yml 中定义错误信息
    flash[:success] =  I18n.t("web.messages.success")
  else
    flash[:error] =  I18n.t("web.messages.failed")
  end
    
end
```

# 自定义 validate 该如何定义错误信息
```ruby
class User < ActiveRecord::Base
  validate :file_size
  
  private
  
  def file_size
    if avatar.file.size.to_f/(1000) > 500
      errors[:avatar] << I18n.t("activerecord.errors.models.user.attribuets.avatar.too_long")
    end
  end
end
  
```

为何在这里调用 errors[:avatar]，错误信息就会放到 @user 中呢？

因为`validate`是在`@user`进行`save`时，才会调用，可以说是实例方法
```ruby
  def file_size
    p self   => 会发现其实是当前实例 @user
  end
```

这种方式只是简单的将错误信息放入`errors`，而且在`yml`并不能够获取`%{attribute}参数`
```ruby
 errors[:avatar] << I18n.t("activerecord.errors.models.user.attribuets.avatar.too_long")
```

我们来更进一步：
```ruby
def file_size
  if avatar.file.size.to_f/(1000) > 500
    # 这样就会调用到`model`的错误信息 (:column, :event_key)
    # 也不用写那么长了
    errors[:avatar] << errors.generate_message(:avatar, :too_long)
  end
end
```

#移除column错误产生的`field_with_errors`
在 form 表单中，如果某个 column 有相关的 errors 信息， Rails 会自动在该`column`相关的元素父层加多一个 div


```ruby
<div class="field_with_errors">
  <label for="job_client_email">Email: </label>
</div>
<div class="field_with_errors">
  <input type="email" value="wrong email" name="job[client_email]" id="job_client_email">
</div>
```

重写掉`config/application.rb`
```ruby
# 原先
@field_error_proc = Proc.new{ |html_tag, instance|
  "<div class=\"field_with_errors\">#{html_tag}</div>".html_safe
}

# 重写后(当前也可以根据需求自己改造)
config.action_view.field_error_proc = Proc.new { |html_tag, instance| 
  html_tag
}
```