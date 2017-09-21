---
layout: post
title: begin and rescue 异常处理机制示例
date:   2016-05-09 14:37:06
categories: ruby
tag: ruby
image: /assets/images/post.jpg
---



##### 使用异常处理机制会停止本次block代码的执行，但是本次block之后的代码可以继续执行。不使用异常处理，遇到异常时就会停止，并且之后的代码不会执行

```

# begin
#   # Raise an error here
#   raise "Error!!"
# rescue
#   #handle the error here
# ensure
#   p "=========inside ensure block"
# end

```

```

  # ary = [1,nil,2,3,4]
  # ary.each do |i|
  #   puts i - 1
  # end

```

##### 1. 异常保存到变量

```

begin
  ary = [1,nil,2,3,4]
  ary.each do |i|
    puts i - 1
  end
rescue => e  #异常保存到变量
  puts e
end

```

##### 2. 用rescue捕获异常

```

begin
  ary = [1,nil,2,3,4]
  ary.each do |i|
    puts i - 1
  end
rescue NoMethodError
  puts "Method Error"
end

```

##### 3. raise抛出异常

```

begin
  ary = [1,nil,2,3,4]
  ary.each do |i|
    raise "The i is nil" if i.nil?
    puts i - 1
  end
rescue ArgumentError
  puts "ArgumentError"
end

```

###### 4. 创建异常类

```

class ThrowExceptionLove < Exception
  puts "Some Error"
end

begin
  raise ThrowExceptionLove, "Got Error"
rescue ThrowExceptionLove => e
  puts "Error: #{e}"
end

```


##### Exception 是异常的总类

###### Ruby提供了一个很好的机制来处理异常。我们附上的代码可以在开始结束块引发的异常，并使用救援条款告诉Ruby我们要处理的异常类型.
###### 执行和异常总是要同时。如果您正在打开的文件不存在，那么，如果你没有正确处理这种情况，那么你的程序被认为是质量很差.

######  如果发生异常，程序停止。所以异常被用来处理各种不同类型的程序的执行过程中可能出现的错误，并采取适当的行动，而不是完全停止程序的.