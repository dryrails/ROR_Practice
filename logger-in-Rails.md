## 简记Rails中的logger实用技巧

##### Gem lograge

[lograge](https://github.com/roidrage/lograge) is awesome for formating log.

```ruby

# Gemfile
gem "lograge"
```

```ruby
# config/initializers/lograge.rb
# OR
# config/environments/production.rb

Rails.application.configure do
  ...
# Logger
  config.lograge.enabled = true
  config.lograge.ignore_actions = ['home#welcome']
  config.lograge.custom_options = lambda do |event|
    params = event.payload[:params].reject do |k|
      ['controller', 'action'].include? k
    end
    options = {
      params: params,    # add params to lograge
      time:   event.time # add time to lograge
    }
    options
  end
  ...
end

```

`lograge` 能帮你格式化日志输出，定义日志输出的格式和内容。去掉了view层渲染的日志内容。如果你不需要日志具体打印view层的渲染内容，用`lograge`是极好的，如果你需要，`lograge` 就不适合你。一般应用不关心view层渲染的具体日志内容。

例子:

环境：Rails 5.1.4

`lograge`还提供了不同的formatters。

```
Lograge::Formatters::Lines.new
Lograge::Formatters::Cee.new
Lograge::Formatters::Graylog2.new
Lograge::Formatters::KeyValue.new  # default lograge format
Lograge::Formatters::Json.new
Lograge::Formatters::Logstash.new
Lograge::Formatters::LTSV.new
Lograge::Formatters::Raw.new       # Returns a ruby hash object
```

默认的日志格式是 key=value的格式：

```ruby
[2017-12-01 22:43:00] [INFO] [] [localhost] [127.0.0.1] [1593754b-bc95-42] method=GET path=/ format=*/* controller=HomeController action=welcome status=200 duration=472.00 view=0.00 params={} time=2017-12-01 11:49:09 +0800

```

比如，配置 Lograge::Formatters::Json.new

```ruby
config.lograge.formatter = Lograge::Formatters::Json.new
```

得到的日志结果是这样的：

```ruby
[2017-12-01 22:43:34] [INFO] [] [localhost] [127.0.0.1] [74cc1bef-2155-42] {"method":"GET","path":"/","format":"*/*","controller":"HomeController","action":"welcome","status":200,"duration":3.09,"view":0.0,"params":{"home":{}},"time":"2017-12-01 22:43:34 +0800"}
```

```ruby
config.lograge.formatter = Lograge::Formatters::Graylog2.new
```

得到的日志结果是这样的：

```ruby
[2017-12-01 22:47:46] [INFO] [] [localhost] [127.0.0.1] [18654bb8-01ec-48] {:_method=>"GET", :_path=>"/", :_format=>"*/*", :_controller=>"HomeController", :_action=>"welcome", :_status=>200, :_duration=>368.34, :_view=>0.0, :_params=>{"home"=>{}}, :_time=>2017-12-01 22:47:46 +0800, :short_message=>"[200] GET / (HomeController#welcome)"}
```

如果想使用

```ruby
config.lograge.formatter = Lograge::Formatters::Logstash.new
```

需要配合安装 `logstash-event`

```ruby
gem "logstash-event"
```

这样很方便的能够得到你想要的日志内容结构，并且被各种日志监控系统兼容。这是`lograge`简单的功能，更多配置查看[文档](https://github.com/roidrage/lograge)

##### 自定义生成log文件

有时候，我们想要将日志存储到一个新的log文件中，和主日志文件区分开来。

定义一个类方法或全局的方法：

```ruby
def create_my_logger(file_name)
  logger = Logger.new("#{Rails.root}/log/#{file_name}")  # 根据file_name，创建一个logger实例。会在Rails.root/log目录下生成file_name文件，用来记录日志
  logger.level = Logger::DEBUG  # 设置日志的level
  logger.formatter = proc do |severity, datetime, progname, message| # 设置日志的formatter
    "[#{datetime.to_s(:db)}] [#{severity}] #{message}\n"
  end

  logger
end
```

在想要使用该日志实例的代码中：

```ruby
log = create_my_logger("third_api_service.log")

... # do something
result = do_third_api_service
if result
  log.info("Third api service success")
else
  log.error("Third api service fail: #{result}")
end
```

个人认为这种方法适用于调用一些关键服务或操作，想要快速了解该服务的正确性，当有错误的时候，可以快速查阅日志，了解服务信息。也便于调试和快速排查问题

还可以给logger实例扩展一些类方法：

```ruby
module MyLogger
  def log_json
    info = {}
    title = []

    yield msg, info

    timestamp  = Time.now.strftime('%Y%m%d_%H%M%S_%L')
    msg      = [timestamp, msg].flatten.compact
    title = title.join('-')

    data = {}
    data[:msg] = msg
    data.merge! info

    self.debug data.to_json # 打印json日志内容
  end
end

def create_my_logger(file_name)
  logger = Logger.new("#{Rails.root}/log/#{file_name}")  # 根据file_name，创建一个logger实例。会在Rails.root/log目录下生成file_name文件，用来记录日志
  logger.level = Logger::DEBUG  # 设置日志的level
  logger.formatter = proc do |severity, datetime, progname, message| # 设置日志的formatter
    "[#{datetime.to_s(:db)}] [#{severity}] #{message}\n"
  end
  logger.extend MyLogger

  logger
end
```

使用示例：
```ruby
SYNC_LOGGER = create_my_logger('sync.log')

SYNC_LOGGER.log_json do |msg, info|
  msg << 'user_id:1'
  info[:action] = 'sync success'
end
```

生成sync.log文件会打印以下信息：

```ruby
[2017-12-01 22:45:20] [DEBUG] {"title":"20171202_141419_311-user_id:1","action":"sync success"}
```

##### 为调用外部服务增加必要日志

当系统中有需要http client形式调用外部服务时，比如：微信开发的微信接口，推送接口，第三方开放API，我会这么做：

使用示例：
```ruby
API_LOGGER = create_my_logger('xxx_api.log')

API_LOGGER.log_json do |msg, info|
  begin
    msg << 'xxx:api:user_id:1'
    info[:resquest] = request_params
    start = Time.now
    res = httpclient.do(url, request_params)
    duration = Time.now - start
    info[:response] = res
    info[:duration] = duration
    info[:error] = nil
  rescue => e
    info[:res] = nil
    info[:error] = "#{e.inspect}"
  end
end
```

将`请求参数`、`返回响应`、`报错信息`都记录下来，查看日志时一目了然。出问题时，能帮助你很快的定位出是系统中的bug还是外部服务的bug。

证明"代码的清白"，就靠他们了。

一个好的日志设计和使用习惯，是一个系统稳定、健壮和可控的重要保证。

本文简单小结了在Rails项目中日志的使用。
