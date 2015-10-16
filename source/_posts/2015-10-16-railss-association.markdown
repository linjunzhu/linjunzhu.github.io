---
layout: post
title: "Rails's association"
date: 2015-10-16 13:42:12 +0800
comments: true
categories: 
---
#Rails 中的关联

##前言
這篇文章是好久好久好久之前所做的笔记，Po 上来吧

## 多对多
1、使用 `has_and_belongs_to_many`

```ruby
class Teacher < ActiveRecord::Base
    has_and_belongs_to_many :students
end

class Student < ActiveRecord::Base
    has_and_belongs_to_many :Teacher
end

# 手动创中间表(migration，注意该表没有主键)
create table :teachers_students, id: false do |t|
    t.integer :teacher_id
    t.integer :student_id
end
```

这种方式是最简单的，不依靠第三个 model 来进行关联。但也仅仅局限于此，其他复杂的关联的都干不了，所以一般不会去用这关联

2、使用 `has_many  :through`
```ruby
# 用户
class User < ActiveRecord::Base
  has_many :comments
  # 如果用户不需要找到帖子, 那么可以不用写关联
  has_many :posts, through: :comments
end
 
# 帖子
class Post < ActiveRecord::Base
  has_many :comments
  has_many :users, through: :comments
end

# 评论
class Comment < ActiveRecord::Base
  belongs_to :posts
  belongs_to :users
end
```

在这里，用户与帖子就是`多对多`的关系，通过`comment`这 model 来进行关联。

总结：
当需要做`数据验证``回调``关联模型需要用到其他属性``关联需要作为一个独立实体`则要用`has_many :through`。

但 `has_and_belongs_to_many`已经不建议使用了，因此直接用`has_many :through`便是。

##一对一
1、`has_one :through` 是个挺好用的特性。

Example: 每个供应商有一个账号，每个账号有一个历史账号
```ruby
# 供应商
class Supplier < ActiveRecord::Base
  has_one :account
end
 
# 账号
class Account < ActiveRecord::Base
  belongs_to :supplier
  has_one :account_history
end

# 历史账号
class AccountHistory < ActiveRecord::Base
  belongs_to :account
end
```

如果是这么写的话，只能这么操作：
```ruby
# 供应商通过账号来查找历史账号
supplier.account.account_history
```

加上 `through`:
```ruby
# 供应商
class Supplier < ActiveRecord::Base
  has_one :account
  has_one :account_history, through: account
end
 
# 账号
class Account < ActiveRecord::Base
  belongs_to :supplier
  has_one :account_history
end

# 历史账号
class AccountHistory < ActiveRecord::Base
  belongs_to :account
end
```
这样供应商直接找到历史账号：
```ruby
supplier.account_history
```

## 多态关联
有时我们会有这样的需求：帖子需要图片的关联，评论需要图片的关联

```ruby
class Picture < ActiveRecord::Base
  belongs_to :imageable, polymorphic: true
end
 
class Post < ActiveRecord::Base
  has_many :pictures, as: :imageable
end
 
class Comment < ActiveRecord::Base
  has_many :pictures, as: :imageable
end
```
这样图片这个model就可以同时属于`Post`以及`Comment`了。
```ruby
# 获取子对象
post.pictures
comment.pictures

# 获取父对象
picture.post
```

Migration 如下：
```ruby
class CreatePictures < ActiveRecord::Migration
  def change
    create_table :pictures do |t|
      t.string  :name
      # 外键id
      t.integer :imageable_id
      # 具体是属于哪个父对象
      t.string  :imageable_type
      t.timestamps
    end
  end
end
```

## 别名

####1、has_many :through
有时我们关联的 model 名会跟想要使用的名称不一样，比如： Genre、 App 与关联类 GenreRecommend  （分类，应用，分类下的推荐应用)。
```ruby
class Genre < ActiveRecord::Base
  has_many :genre_recommends
  # 找到该分类下的推荐 apps
  has_many :apps, through: :genre_recommend
end

class App < ActiveRecord::Base
  has_many :genre_recommends
  # 这里并不需要找到 genre, 因此可以不写
  has_many :genres, through: :genre_recommend
end

class GenreRecommend < ActiveRecord::Base
  belongs_to :genre
  belongs_to :app
end
```

```ruby
# 一般使用（但 apps 有可能会跟其他冲突, 比如找出该分类下的所有 apps）
@genre.apps

# 实际上这么使用会更友好
@genre.recommends
```

修改后：( 通过 `source` )
```ruby
class Genre < ActiveRecord::Base
  has_many :genre_recommends
  # 找到该分类下的推荐 apps
  has_many :recommends, source: :app, through: :genre_recommend
end

class App < ActiveRecord::Base
  has_many :genre_recommends
end

class GenreRecommend < ActiveRecord::Base
  belongs_to :genres
  belongs_to :app
end
```

####2、`belongs_to``has_one``has_many`

这三者都是通过`:class_name`来实现别名的。

```ruby
class Customer < ActiveRecord::Base
  has_many :orders, class_name: "Transaction"
end
```

我猜是因为`source`主要是跟`through`的第三个 model 中`belongs_to`对应

而`class_name`就仅仅只是指定类名了。

## 关联删除
`一对多`关系中，往往删除主对象，我们还需要顺便删除所有子对象的需求。

Example: 一个人拥有几个联系方式

```ruby
class Person < ActiveRecord::Base
  # 这样删除一个 person 时，还会删除名下的所有 contacts
  has_many :contacts, dependent: :destroy
end

class Contact < ActiveRecord::Base
  belongs_to :person
end
```


## 自动保存对象
```ruby
class Person < ActiveRecord::Base
  has_many :contacts, dependent: :destroy
end

class Contact < ActiveRecord::Base
  belongs_to :person
end
```
```ruby
@person = Person.first
c1 = Contact.new
c2 = Contact.new

# 此时会自动保存 c1 c2，官方说因为 c1 c2 要更新外键 ( 这点不太明 )
@person.contacts << c1
@person.contacts << c2
```

将对象赋给`has_many``has_one`的主对象都会被自动保存。

当然，主对象必须是有保存过的才行。（存在于数据库)

## counter_cache

Example: 

帖子下有多个评论，显示时经常需要显示评论的个数。评论的个数经常会变动，如果直接`post.comments.size`难免效率会低。

```ruby
class Post < ActiveRecord::Base
  has_many :comments
  # 只读，否则被串改就不好了。
  attr_readonly :comments_count
end

class Comment < ActiveRecord::Base
  belongs_to :post, counter_cache: true
end
```
创建字段
```ruby
def change
  add_column :posts, :comments_count, :integer
end

```

这样声明后，Rails 会及时更新缓存，调用 size 方法时，会返回缓存中的值 ( 官方 guide 是说缓存，不过我认为仅仅只是将值放在该 model 的字段中，并无缓存 )
```ruby
@post.comments.size
```

## 点击数、阅读数
文章的`阅读数``点击数`也是经常变化，有无更好的方法呢？例如存入缓存？

可以使用我自己写的一个 GEM

[https://github.com/linjunzhu/counter-cache-credis](https://github.com/linjunzhu/counter-cache-credis)