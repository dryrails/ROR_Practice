---
layout: post
title: 10 most underused activerecord relation methods(翻译)
date:   2016-05-19 15:26:06
categories: rails
tag: rails AR::Relation
image: /assets/images/post.jpg
---



### 10.first_or_create with a block
常用的first_or_create:

```ruby
Book.where(title: 'Tale of Two Cities').first_or_create

```

一般情况，你想要通过一个确定的属性找到一个记录，或者用增加的这个属性新建一个记录。
你可以通过block来使用first_or_create:

```ruby
Book.where(:title => 'Tale of Two Cities').first_or_create do |book|
  book.author = 'Charles Dickens'
  book.published_year = 1859
end

```

当你没找到相应数据的时候，这里的操作就相当于:

```ruby
Book.create(title: 'Tale of Two Cities',
            author: 'Charles Dickens',
            published_year: 1859)

```

传的三个属性都将用于create操作

同样的，find_or_create_by 也可以进行block的操作

### 9.first_or_initialize
如果你还不想要save这条记录，可以使用first_or_initialize:

```ruby
Book.where(:title => 'Tale of Two Cities').first_or_initialize

```

和first_or_create 一样，你也可以使用 block方式，增加更多的属性。

### 8.scoped
有时候你想要用 ActiveRecord::Relation 表示一个类的所有数据记录，你可以很容易的使用scoped方法构造出来:

```ruby
def search(query)
  if query.blank?
    scoped
  else
    q = "%#{query}%"
    where("title like ? or author like ?", q, q)
  end
end

```

### 7.none(rails 4 only)
同样的，有时候你想要用 ActiveRecord::Relation 表示没有对象。如果客户的API 期望关系对象，返回一个空数组不是一个号的方法。你可以用 none 代替表示。

```ruby
def filter(filter_name)
  case filter_name
  when :all
    scoped
  when :published
    where(:published => true)
  when :unpublished
    where(:published => false)
  else
    none
  end
end

```

### 6.find_each
如果你想要遍历成千上万的数据记录，用each已经不合适。它会执行一次查询去获得所有的数据记录，然后把它们实例化后存入内存。如果你有足够的内存去使用，可以这么做。否则，这非常容易让Rails app处于负载假死而崩溃。 find_each相反。获取一批数据记录一次默认(1000)和实例化这一次,这样你没有把所有记录实例化存入内存中。

```ruby
Book.where(:published => true).find_each do |book|
  puts "Do something with #{book.title} here!"
end

```

请注意,您不能指定的顺序由find_each记录了。如果你指定一个关系,它会被忽略。(意思似乎是: 无法给记录进行order排序)

### 5.to_sql and explain
ActiveRecord是很棒的,但它并不总是你认为它会生成查询。跳在控制台中,您正在构建的关系上运行这些命令,以确保它映射到一个智能查询,或者使用您制作精良的指标:

```ruby
Library.joins(:book).to_sql
# => SQL query for you database.
Libray.joins(:book).explain
# => Database explain for the query.


```

### 4.find_by(rails 4 only)

```ruby
Book.where(:title => 'Three Day Road', :author => 'Joseph Boyden').first

Book.find_by(:title => 'Three Day Road', :author => 'Joseph Boyden')

```

### 3.scoping
你可以“范围”一个类的方法到一个特定的关系。考虑下面的例子从Rails文档:

```ruby
Comment.where(:post_id => 1).scoping do
  Comment.first # SELECT * FROM comments WHERE post_id = 1
end

```

### 2.pluck  and  ActiveRecord::Base.connection.select_all
想要对某些记录数组的列值吗?使用pluck

```ruby
published_book_titles = Book.published.pluck(:title)

```

想要得到记录以hash的结构形式返回，使用ActiveRecord::Base.connection.select_all

```ruby
ActiveRecord::Base.connection.select_all('SELECT * FROM users')

```

### 1.merge
我不能没有这个宝石,但奇怪的是un-documented源,而不是我见过任何指导或书中提到。其他用途,它可以让你做一个连接,通过命名范围和过滤加入模型:

```ruby
class Account < ActiveRecord::Base
  # ...

  # Returns all the accounts that have unread messages.
  def self.with_unread_messages
    joins(:messages).merge( Message.unread )
  end
end

```

原文: http://www.mitchcrowe.com/10-most-underused-activerecord-relation-methods/
