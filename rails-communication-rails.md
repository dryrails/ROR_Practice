## Rails App之间的三种"通讯"方式实践

### 三种方式
1. API接口通讯
2. Sidekiq
3. gRPC

### Rails环境
+ Rails 5.1.4
+ Ruby 2.3.3
+ redis

Two Rails App:

1. jerry_app

listen: 127.0.0.1:9001

2. tom_app

listen: 127.0.0.1:9002

定义一个请求方向： tom_app -> 发起请求 -> jerry_app

##### 文章Rails App完整的代码在：[完整代码repo](https://github.com/pathbox/Rails_Communicate_Rails)

### API接口通讯

简单、稳定的方式，没有"第三方的依赖"。

在`jerry_app`定义一个对外接口，`get`请求，url是 `http://127.0.0.1/product`， controller的代码：

```ruby
# products_controller.rb
class ProductsController < ApplicationController
  before_action :do_validation, only: :index

  def index
    render json: {code: 200, message: "Success"}
  end

  private

  def do_validation
    timestamp = params[:timestamp].to_i
    token = params[:token]

    if timestamp.blank? || token.blank?
      return render json: {code: 400, message: "timestamp or token can't be blank"}, status: 400
    end

    time_now = Time.now.to_i
    if time_now - timestamp > 300 || time_now < 0
      return render json: {code: 400, message: "Timestamp is expired"}, status: 400
    end

    str = APP_ID + timestamp.to_s
    hex_token = OpenSSL::HMAC.hexdigest("SHA1", APP_KEY, str)

    if token != hex_token
      return render json: {code: 401, message: "token is wrong"}, status: 401
    end

  end
end
```

在`tom_app`定义一个方法，请求`jerry_app`开放的接口：

```ruby
# tom_app/app/services/product_service.rb
class ProductService

  class << self
    def get_products
      timestamp = Time.now.to_i
      str = APP_ID + timestamp.to_s
      token = OpenSSL::HMAC.hexdigest("SHA1", APP_KEY, str)

      params = "token=#{token}&timestamp=#{timestamp}"
      url = "127.0.0.1:9001/products?#{params}" # jerry_app listens 9001

      resp = Typhoeus.get(url, timeout: 3)
      body = resp.body
      puts "Products result: #{body}"
    end
  end
end
```

接口进行了参数验证和token方式的权限认证，保证一定的安全性。实际生产环境使用https会更好。

鉴权的算法是：

```ruby
str = APP_ID + timestamp.to_s
token = OpenSSL::HMAC.hexdigest("SHA1", APP_KEY, str)
```

然后对token进行了过期验证：

```ruby
time_now = Time.now.to_i
if time_now - timestamp > 300 || time_now < 0
  return render json: {code: 400, message: "Timestamp is expired"}, status: 400
end
```

你还可以用IP白名单的方式，让接口只允许白名单的IP访问。

### 使用 Sidekiq "通讯"

其实这是一种使用消息队列，进行异步调用的方式。并不是只有`Sidekiq`能做到，一般的消息队列框架都可以实现，简单的使用redis也可以实现。

在Rails(Ruby)环境，顺其自然的，就选择了`Sidekiq`。分别在两个Rails App进行Sidekiq操作，代码示例：

```ruby
# tom_app/config/initializers/tom.rb

jerry_redis_config = {
  host: '127.0.0.1',
  port: 6379,
  db: 0
}

jerry_redis = Redis.new(jerry_redis_config)

jerry_sidekiq_redis = Redis::Namespace.new(:jerry_app, redis: jerry_redis)

JERRY_REDIS_POOL = ConnectionPool.new(timeout: 1) {jerry_sidekiq_redis}

```

上面我们定义了一个全局的　`ConnectionPool`实例对象`JERRY_REDIS_POOL`。使用到的redis配置是我们要调用的`jerry_app`的redis配置：`jerry_redis_config`。
也就是`JERRY_REDIS_POOL`是`jerry_app`的`ConnectionPool`实例对象。

然后我们定义一个Sidekiq Worker：

```ruby
# tom_app/app/workers/update_product_worker.rb

class UpdateProductWorker
  include Sidekiq::Worker
  sidekiq_options :queue => :default, :pool => JERRY_REDIS_POOL
end
```

在sidekiq_options 中配置 pool的值为JERRY_REDIS_POOL。在这里不需要定义`perform`方式。

在你想要向`jerry_app`发起调用时，调用`UpdateProductWorker`就行。

```ruby
# tom_app/app/services/product_service.rb
class ProductService

  class << self
    # ......

    def update_product_worker
      UpdateProductWorker.perform_async(1)
    end
  end
end
```

接下来在`jerry_app`也定义相同名称的`UpdateProductWorker`

```ruby
# jerry_app/app/services/product_service.rb
class UpdateProductWorker
  include Sidekiq::Worker
  sidekiq_options :queue => :default

  def perform(product_id)
    # do something for example:
    # product = Product.find(product_id)
    # product.update!(amount: 0)
    Sidekiq.logger.info "++++++product_id: #{product_id}"
    Sidekiq.logger.info "UpdateProductWorker Success"
  end
end
```

在`tom_app`中打开`rails c`:

```ruby
ProductService.update_product_worker
```

在`jerry_app`中，`tailf sidekiq.log -n 20`得到了：

```ruby
2018-01-05T05:55:07.681Z 28855 TID-hooyg UpdateProductWorker JID-2714364eff0c2d6714d38221 INFO: start
2018-01-05T05:55:07.683Z 28855 TID-hooyg UpdateProductWorker JID-2714364eff0c2d6714d38221 INFO: ++++++product_id: 1
2018-01-05T05:55:07.683Z 28855 TID-hooyg UpdateProductWorker JID-2714364eff0c2d6714d38221 INFO: UpdateProductWorker Success
2018-01-05T05:55:07.683Z 28855 TID-hooyg UpdateProductWorker JID-2714364eff0c2d6714d38221 INFO: done: 0.002 sec
```

原理： tom_app 是"消费者"， jerry_app是"生产者"。借助Sidekiq，"消费者"得到参数并执行具体的逻辑。显然，这是一种异步请求调用。如果你需要同步的请求调用，这种方式就不合适了。

### gRPC in Ruby on Rails

如果不了解gRPC，可以阅读这里[gRPC官网](https://grpc.io/)，是一种rpc的方式。在之前Go的微服务项目中，服务之间使用了gRPC进行通讯。
gRPC对ruby也有支持，[gRPC ruby example](https://github.com/grpc/grpc/tree/master/examples/ruby)。阅读了它的例子和文档，觉得实现方式和Rails并没有融合的很好。
然后在github上搜了一下，找到了这个gem: [gruf](https://github.com/bigcommerce/gruf)。它是将gRPC ruby 进行了一定的封装。于是我就选择了使用它。不过我发现，`gruf`的`README`
在一些细节上不够完整，有些坑，我按照README的操作，没有成功。直到把他们的demo的代码看完，和demo中rake 测试命令的代码看完，才明白了所有细节。我修改了README，并给他们提了push。
如果你在本地没法跑通例子，也许你需要安装gRPC 和 protocol buffers。

看具体的代码例子，主要的代码逻辑在`app/rpc`目录下：

`tom_app`中

```ruby
# tom_app/config/initializers/gruf.rb

require 'gruf'
require 'app/proto/helloworld_services_pb'

Gruf.configure do |c|
  c.default_client_host = '127.0.0.1:9003' # 在这里默认配置了host，如果不配置，则需要在每次调用的时候传host值
end

# 在 tom_app/app/rpc/app/ 执行 grpc_tools_ruby_protoc -Iproto --ruby_out=proto --grpc_out=proto proto/helloworld.proto 会生成　helloworld_pb.rb和helloworld_services_pb.rb文件。这样有一个问题，就是helloworld_pb 文件的加载问题，应该和加载路径有关。我就手动修改了 helloworld_services_pb.rb文件

require 'app/proto/helloworld_pb' # 这样就正确加载了helloworld_pb.rb了

# 这里不列出来了，具体的client端和server端，就是使用helloworld_pb.rb和helloworld_services_pb.rb文件中的类和方法

# tom_app/app/rpc/greeter_client.rb
class GreeterClient

  def self.say_hello(name)
    puts "say_hello: #{name}"

    options = {
      # hostname: '127.0.0.1:9003',
      username: 'admin',
      password: 'admin'
    }

    begin
      client = ::Gruf::Client.new(service: ::Helloworld::Helloworld, options: options)

      response = client.call(:SayHello, name: name)
      puts "+"*30
      puts response.message.message # Helloworld::HelloReply instance
    rescue Gruf::Client::Error => e
      puts e.error.inspect
    end
  end
end
```

`tom_app`中

```ruby
# tom_app/config/initializers/gruf.rb
require 'gruf'
require 'app/proto/helloworld_services_pb'

Gruf.configure do |c|
  c.server_binding_url = '127.0.0.1:9003'
  c.interceptors.use(Gruf::Interceptors::Instrumentation::RequestLogging::Interceptor, formatter: :logstash)
  # basic auth
  c.interceptors.use(
    Gruf::Interceptors::Authentication::Basic,
    credentials: [{
      username: 'admin',
      password: 'admin',
    },{
      username: 'another-username',
      password: 'another-password',
    },{
      password: 'a-password-only'
    }]
  )
end

# helloworld.proto
# helloworld_pb.rb
# helloworld_services_pb.rb
# 这三个文件和　tom_app一样

# jerry_app/app/rpc/greeter_controller.rb
class GreeterController < Gruf::Controllers::Base
  bind ::Helloworld::Helloworld::Service

  def say_hello
    name = request.message.name
    result = "+++ Hello #{name}+++"
    puts result
    Helloworld::HelloReply.new(message: result)
  end
end
```

在`jerry_app`根目录执行`bundle exec gruf`,你可以看到
```ruby
[2018-01-05T15:31:46.719862 #31505]  INFO -- : handling /helloworld.Greeter/SayHello with #<Method: Helloworld::Hello::Service#say_hello>
```

在`tom_app`　rails c 中执行

```ruby
2.3.3 :001 > GreeterClient.say_hello("Tom")
say_hello: Tom
D, [2018-01-05T15:34:15.095475 #32363] DEBUG -- : calling 127.0.0.1:9003:/helloworld.Greeter/SayHello
++++++++++++++++++++++++++++++
+++ Hello Tom+++
 => nil
```

在`jerry_app　gruf` 服务，你可以看到这样的日志

```ruby

+++ Hello Tom+++
I, [2018-01-05T15:35:49.945836 #31505]  INFO -- : {"message":"[GRPC::Ok] (helloworld.hello.say_hello)","status":0,"service":"helloworld.hello","method":"say_hello","action":"say_hello","grpc_status":"GRPC::Ok","duration":0.74,"thread_id":40308860,"time":"2018-01-05 15:35:49 +0800","host":"20161125","format":"json"}
```

整个过程：将gRPC client端实现为一个类，定义调度的类方法或实例方法，使用了账号密码的basic auth；在gRPC server端具体实现要调度的方法代码逻辑。
你可以查阅更多`gruf`的内容，它实现了其他一些中间件(interceptor),让你更好的使用gRPC。你可以选择用[foreman](https://github.com/ddollar/foreman)来控制`gruf`服务的启动。

### 其他
我用go-grpc的作为client，调用`jerry_app`的`gruf`服务，结果报了错。
```ruby
W, [2018-01-05T15:59:22.046980 #31505]  WARN -- : UNIMPLEMENTED: #<struct Struct::NewServerRpc method="/proto.Hello/SayHello", host="127.0.0.1:9003", deadline=1970-01-01 07:59:59 +0800, metadata={"user-agent"=>"grpc-go/1.8.0-dev"}, call=#<GRPC::Core::Call:0x000000045a7e50>>
```

关于跨语言的rpc调用，我还未尝试。[thrift](https://github.com/apache/thrift)应该会是一个不错的选择。
