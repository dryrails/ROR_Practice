---
layout: post
title: 为rails项目升级使用Elasticsearch 5.x版本
date:   2017-10-14 21:01:06
categories: Elasticsearch
image: /assets/images/elasticsearch.png
---

最近为项目系统中使用的Elasticsearch进行了升级。从原来1.4版本的Elasticsearch升级到5.x版本。

由于5.x版本的Elasticsearch不再支持1.4版本中的很多查询语法，所以不可避免的进行了查询层的重构。
每次Elasticsearch有重大版本升级的时候，会在官方文档中列出相应的改变，你需要认真阅读的`Breaking changes`内容。

最重要的事情是，终于可以利用这次机会，为Elasticsearch加入了索引别名机制

##### 安装

Elasticsearch-5.x server 版本的安装可以参考之前的文章:。我是安装在Ubuntu16.04上，Ubuntu14.04也是可以的。

在`Gemfile`中增加

```ruby
gem "elasticsearch"
gem "elasticsearch-model"
gem "elasticsearch-rails"
```

使用最新的elasticsearch gem包，至少大于5.0版本

##### 加入索引别名机制
为什么说别名机制是最重要的事情呢？我们知道Elasticsearch的索引，当你配置并建立好之后，是无法进行更新修改的。
想要进行更新修改，需要删除原来的整个索引，然后再根据新的配置，重建索引，包括索引的文档数据。这就不可避免了该索引在重建过程中
有一段时间是不可用的。

举个简单的例子：Elasticsearch有一个orders索引，大概有1000w条记录数据。你的客户会经常使用查询orders的功能，这个功能查的就是orders索引。
现在你要修改orders索引的分词器。这时你需要重建orders索引(删除再新建)，假设这个过程，让orders索引完全恢复到可用状态需要10分钟。那么，你的客户就会
有10分钟没法使用orders的搜索功能。当然，你可以在凌晨3点进行这个操作。这里只是举个例子。Elasticsearch索引别名机制可以帮助你进行索引`无缝`的升级或更新。

##### 别名机制的过程及原理
关键的操作是：取别名。 现在我要重构orders这个索引，我会先创建一个叫做orders_v1的索引，为其配置名称为orders的别名。
这样，在查询代码层，你就可以使用类似Order.__elasticsearch__.search() 这样的查询了。其实，也就是 在代码层面上，不知道有orders_v1这个索引额存在，
一切对Elasticsearch的操作，都通过别名orders操作。

现在，我们需要修改orders_v1索引，只要简单的三步:

+ 1. 用新的索引配置新建一个叫orders_v2的索引，将文档数据也建立好
+ 2. 进行切换别名的操作，将orders别名由原来order_v1指向orders_v2
+ 3. 看你的需要，再进行增量变更数据的更新

切换别名的操作非常快，并且你的代码层的修改会变得很轻松。几乎不会对用户的使用造成影响。

简单的图表示：

![orders]( /assets/images/orders.png "Orders")

在项目中，实现别名机制不难。详见[官方文档的例子](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html)

新建索引别名
```
POST /_aliases
{
    "actions" : [
        { "add" : { "index" : "order_v1", "alias" : "orders" } }
    ]
}
```

切换索引别名
```
POST /_aliases
{
    "actions" : [
        { "remove" : { "index" : "order_v1", "alias" : "orders" } },
        { "add" : { "index" : "order_v2", "alias" : "orders" } }
    ]
}
```


在rails项目中，我通过rake脚本就实现，下面是我写的一个rake脚本，仅供参考。

```ruby
#coding: utf-8
client = Elasticsearch::Client.new(
  host: elasticsearch_url,
  retry_on_failure: 0,
  log: true,
  transport_options: { request: { timeout: 10 } }
  )

# ++++++++++++++++++++++++++++++++++使用方式++++++++++++++++++++++++++++++++++++++

#1 创建索引 　　 # bundle exec rake es_action:create_index table=Order index_name=orders_v1
#2 索引别名配置　#　bundle exec rake es_action:update_index_alias alias=orders new_index=orders_v1 old_index=orders_v2(可选，交换索引的时候使用)
namespace :es_action do

  desc "重建es索引,根据传入的参数指定重置的索引"
  # 索引名称会是 es_name_日期,比如:tickets_2017_09_26
  task create_index: :environment do
    table = ENV['table']
    index_name = ENV['index_name']
    puts "++++++ start remapping index #{table}"
    if table.blank?
      puts "model 参数不能为空"
      return
    end

    model = table.camelize.constantize

    model.__elasticsearch__.create_index! index: index_name, force: true  # ++++++创建索引
    model.__elasticsearch__.refresh_index! index: index_name
  end

  desc "新建或交换es的别名（将别名由旧的索引指向新的索引）"
  task update_index_alias: :environment do
    old_index = ENV['old_index']
    new_index = ENV['new_index']
    alias_name = ENV['alias']

    puts "++++++ start update_index_alias #{alias_name}"
    if alias_name.blank? || new_index.blank?
      puts "++++++ alias new_index 参数不能为空"
      return
    end

    body = {actions: []}
    body[:actions] << {remove: {index: old_index, alias: alias_name}} if old_index.present? # old_index存在，表示是进行别名切换
    body[:actions] << {add: {index: new_index, alias: alias_name}}

    client.indices.update_aliases body: body

    puts "++++++ update_index_alias success"
  end

end

```

##### mapping的配置

`app/models/order.rb`

1.4版本中的mapping配置
```ruby
indexes :subject, type: 'multi_field' do
  indexes :raw, type: :string, index: :not_analyzed
  indexes :tokenized, analyzer: :ik_smart
end
indexes uuid, type: :string
indexes amount, type: :integer
indexes price, type: :long
indexes :created_at, type: :date
indexes :updated_at, type: :date
indexes :users do
  indexes :id, type: :integer
  indexes :name, type: :string, index: :not_analyzed
end
```

5.x版本中的mapping配置

```ruby
indexes :subject, type: :keyword do
  indexes :raw, type: :keyword
  indexes :tokenized, analyzer: :ik_smart
end
indexes uuid, type: :text
indexes amount, type: :integer
indexes price, type: :long
indexes :created_at, type: :date
indexes :updated_at, type: :date
indexes :users do
  indexes :id, type: :integer
  indexes :name, type: :keyword
end
```

5.x版本没有了string类型，拆分成了 keyword和text。如果你指定了analyzer，则是text类型。

不再有 type: 'multi_field'， 这个好像是2.x版本就进行了修改

5.x的IK也分成了两种配置。 `ik_smart`和`ik_max_word`， `ik_max_word`拥有更细的分词粒度。

##### 查询语句的重构

[官方文档的query changes](https://www.elastic.co/guide/en/elasticsearch/reference/5.0/breaking_50_search_changes.html)

修改例子：

##### 不再使用filtered
```ruby
{filtered: {
  filter: {
    bool: {
      must: [{or: [{terms: {user_id: user_id}}]},
      {term: {order_id: order_id}}
     ],
    }
   }
 }
}

# 重构为：
{bool: {
  must:[
    {term: {order_id: order_id}},
  ],
  should: [
    {terms: {user_id: user_id}}
  ]
}}


```

##### 不再使用or
```ruby
filters[:must] << {
  or: [
    {term: {order_id: order_id}},
    {term: {user_id: user_id}}
  ]
}

#重构为:
filters[:must] << {
  bool: {
    should: [
      {term: {order_id: order_id}},
      {term: {user_id: user_id}}
    ]
  }
}
```

##### 不再使用missing
```ruby
{missing: {field: field_name}}

#重构为：
{"bool": {
   "must_not": [
      {
        "exists":{
           "field": field_name
        }
      }
   ]
  }
}
```

##### execution: 'and'不再使用
```ruby
{terms: {field_name => value, execution: "and"}}

#重构为：
terms_array = value.map {|v| {term: {field_name => v}}}
{bool: {must: terms_array}}
```
