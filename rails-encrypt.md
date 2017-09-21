# The encryption in ruby and rails


在一个项目中会有很多数据是需要加密的，以保证数据不会泄露。
比如: 密码的保存。登入时，cookie保存的值。做支付接口的时候，对参数进行加密后传递等待。
加密的作用就是为了防止他人得到真实的数据，从而进行一些操作，避免用户或项目受到损失。

###### Ruby库中提供的加密方法
Ruby库中提供了比较全的加密方法。比如简单的MD5、SHA1等。

```ruby

require 'digest'
Digest::MD5.digest 'hello world'
# =>  "^\xB6;\xBB\xE0\x1E\xEE\xD0\x93\xCB\"\xBB\x8FZ\xCD\xC3"

Digest::SHA1.digest 'hello world'
# =>  "*\xAEl5\xC9O\xCF\xB4\x15\xDB\xE9_@\x8B\x9C\xE9\x1E\xE8F\xED"

Digest::SHA256.digest 'hello world'
# =>  "\xB9M'\xB9\x93M>\b\xA5.R\xD7\xDA}\xAB\xFA\xC4\x84\xEF\xE3zS\x80\xEE\x90\x88\xF7\xAC\xE2\xEF\xCD\xE9"

```

###### Encoding formats
```ruby
Digest::MD5.hexdigest 'hello world'
#=>  "5eb63bbbe01eeed093cb22bb8f5acdc3"

Digest::MD5.base64digest 'hello world'
#=>  "XrY7u+Ae7tCTyyK7j1rNww=="

Digest::SHA1.base64digest 'hello world'
#=>  "Kq5sNclPz7QV2+lfQIuc6R7oRu0="

Digest::SHA1.hexdigest 'hello world'
#=.  "2aae6c35c94fcfb415dbe95f408b9ce91ee846ed"

Digest::SHA256.hexdigest 'hello world'
#=>  "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"

Digest::SHA256.base64digest 'hello world'
#=>  "uU0nuZNNPgilLlLX2n2r+sSE7+N6U4DukIj3rOLvzek="
```

###### Compute digest by chunks

```ruby
md5 = Digest::MD5.new
md5.update "hello "
md5 << "world"
md5.hexdigest    #=>  "5eb63bbbe01eeed093cb22bb8f5acdc3"
```

###### Compute digest for a file

```ruby
sha256 = Digest::SHA256.file 'testfile'
sha256.hexdigest
#=> "a948904f2f0f479b8f8197694b30184b0d2ed1c1cd2a1ec0fb85d299a192a447"
```

testfile 文件里是 "hello world"字符串。这里只是把这个字符串内容取出来了，然后加密显示出来，并没有真正改变文件的属性。

关于SHA2(Secure Hash Algorithm 2)，其实是这样的: SHA2 family => [SHA256, SHA384, SHA512] 有三种加密方式。这三种方法的区别：简单的说，就是
后面的数字越大，加密得到的结果位数越大，结果也越复杂。

MD5 的加密或是SHA 的加密都是不可逆的加密。就是理论上来说，得到的加密结果(一定位数的随机性字符串)是没法反推回真正的值。而由于计算机的发展，可以用计算机进行不断的
加密得到值，再进行一种配对来得到原始值。像MD5 就已经被大量破解了。如果计算机一直计算运行下去，SHA 也会继续被大量破解，只是需要时间。对他们的加密原理等内容这里就不具体深入了。

所以，MD5的加密方式其实已经不够安全了。如果数据不是安全性很高，可以简单的使用MD5，不然选择SHA256及以上的加密方法，加密效果会更有效。
不过MD5可以得到随机且几乎不相同的字符串(已经证实MD5中有碰撞)。使用MD5 可以构造不重复的随机字符串。比如用来做数据库的唯一索引等。

###### An experimental implementation of HMAC keyed-hashing algorithm

```ruby

# one-liner example
OpenSSL::HMAC.hexdigest('SHA1', "hash key1", "123456")
#=>  "2633f494ca74fe3b08d4f25272b645c166843292"

OpenSSL::HMAC.hexdigest('SHA1', "hash key2", "123456")
#=>  "f5295736f4a6745d413c6ce306d4a640ff5c54a0"

OpenSSL::HMAC.hexdigest('SHA256', "hash key2", "123456")
#=>  "3f0733ba5955b2d0916da2b4a2051ca8c155a1e0a2e7d1df874e4eff6b74b7c4"

# rather longer one
hmac = OpenSSL::HMAC.new("foo", 'SHA1')
#=>  ae782bb10f8186761d3558c2ff8c4535999d6f01

hmac.digest
#=>  "\xAEx+\xB1\x0F\x81\x86v\x1D5X\xC2\xFF\x8CE5\x99\x9Do\x01"

hmac << 'bar'
hmac.update('bar')
#=>  46b4ec586117154dacd49d664e5d63fdc88efb51  it is not a string
hmac.class
#=> OpenSSL::HMAC

hmac.hexdigest
hmac.inspect
hmac.to_s
#=>  "46b4ec586117154dacd49d664e5d63fdc88efb51"
```

从上面的代码我们可以看到，使用的加密方法不同，或者hash key的取值不同，加密的结果不同。
这样就有很灵活的加密方式，而且也是更为复杂的加密方式。
这种加密方法的一个应用场景就是：signature的匹配。很多时候，比如调用一些第三方的外部API的时候，会有signature的
匹配机制。匹配成功了，则继续往下走。像支付接口，微信接口，推送接口，短信接口等等。下面举个简单的例子。

```ruby
timestamp = "1468463483.943864"
token = "3f0733ba5955b2d0916da2b4a2051ca8c155a1e0a2e7d1df874e4eff6b74b7c4"
appkey = '1erv3tw-4tzn-h36y-tdpd-y67kqhqy6s'
params = "#{timestamp}&#{token}"
signature = OpenSSL::HMAC.hexdigest('SHA256', appkey, params)

#=>  "bb3fdfe9c72e3277b8f5fb76a200abd7b5dcdc64c9f8b337f7a10dbf0f91e857"
```

根据API文档要求的加密算法，得到signature，将signature和参数按照要求一起传到第三方的API接口。
第三方API接口服务器会利用参数再进行一次加密计算得到signature，和传过来的signature进行匹对。
有时候，对参数还需要进行一定的编码，比如用Base64。整个流程简单描述就是这样的。而这里大家可以知道，appkey这个
字符串是只有你和API接口服务器知道的，如果泄密了，你的加密算法也就泄密了，别人就能很容易的进行伪造
你的请求并且成功。 所以，别泄露appkey。
