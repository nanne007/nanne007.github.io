---
title: Sinatra 应用的配置
date: 2014-10-09T00:11:45+08:00
---

在 Sinatra 中，经常需要使用 `set`、`enable`、`disable` 这些函数来配置应用，
并在请求上下文通过 `settings` 访问这些配置。

例如：

``` ruby
# example 1
set :author, "XXX"
get "/app/author" do
  "the author is " + settings.author
end
```

也可以利用代码块来设置：

``` ruby
# example 2
set(:author) do
  # do something here
  “XXX”
end
```

同时，可以使用 `Proc` 访问其他设置选项：

``` ruby
# example 3
set :app_info, Proc.new do
  "desc: " + "app about how to use set." + author
end
```

另外，传递一个哈希表给 `set` 函数，可以一次配置多个选项：

``` ruby
# example 4
set author: "XXX", app_info: Proc.new { "desc: " + "app about how to use set." + author }
```

初次使用这些的时候，时常会被下面这样的问题困扰着。
- `settings` 是什么？
- 为什么在 `settings` 上可以访问用 `set` 方法设置的选项？
- 为什么在请求上下文要通过 `settings` 访问配置选项，而不同配置选项却可以互相访问，正如 **example 3** 所做的那样？

要想解决这些问题，首先需要去了解 Sinatra 的作用域。
## Sinatra 的作用域

大家都知道，Ruby 的作用域门决定了不同作用域下所能访问的方法和变量。

“[scopes and binding in sinatra readme]("http://www.sinatrarb.com/intro.html#Scopes%20and%20Binding")” 介绍了 Sinatra 的三种不同的作用域。
- 应用层作用域或者说类作用域（Application/Class Scope）
- 请求作用域或者说实例作用域（Request/Instance Scope）
- 委托作用域（Delegation Scope）

下面逐一介绍这三个作用域。
### 应用层作用域

每个 Sinatra 应用都是 `Sinatra::Base` 的子类。
因此，`Base` 的类方法，比如 `get`, `before`, `set`，都可以在子类作用域中使用。

``` ruby
require 'sinatra/base'
class MyApp < Sinatra::Base
  set :author, "XXX"
  get "/app_info" do
    "app name: MyApp"
  end
end
```
### 请求作用域

每一个请求都会有一个实例与之对应，处理逻辑都是在这个实例的上下文中进行的。
在这个作用域中，可以使用 `Base` 的实例方法，像 `settings`, `halt`, `pass`，
以及各种属性，像 `request`, `response`, `params`。
由于 `Base` 混入了 `Helpers` 和 `Templates` 模块，
也可以使用 `redirect`, `uri` 以及 `erb` 这些辅助方法。

``` ruby
require 'sinatra/base'
class MyApp < Sinatra::Base
  set :author, "XXX"
  get "/app_info" do
    # here comes request scope
    "app name: MyApp" + settings.author
  end
end
```
### 委托作用域

当使用 `require 'sinatra'` 创建一个经典的 Sinatra 应用时，委托作用域出现了。
委托作用域中的方法被委托给应用层作用域。
源码中，定义了一些被委托的方法。

``` ruby
delegate :get, :patch, :put, :post, :delete, :head, :options, :link, :unlink,
             :template, :layout, :before, :after, :error, :not_found, :configure,
             :set, :mime_type, :enable, :disable, :use, :development?, :test?,
             :production?, :helpers, :settings, :register
```
## set源码

下面是 sinatra-1.4.4 的 `set` 实现：

``` ruby
class Base
  # ...

  class << self
    # Sets an option to the given value.  If the value is a proc,
    # the proc will be called every time the option is accessed.
    def set(option, value = (not_set = true), ignore_setter = false, &block)
      raise ArgumentError if block and !not_set
      value, not_set = block, false if block

      if not_set
        raise ArgumentError unless option.respond_to?(:each)
        option.each { |k,v| set(k, v) }
        return self
      end

      if respond_to?("#{option}=") and not ignore_setter
        return __send__("#{option}=", value)
      end

      setter = proc { |val| set option, val, true }
      getter = proc { value }

      case value
      when Proc
        getter = value
      when Symbol, Fixnum, FalseClass, TrueClass, NilClass
        getter = value.inspect
      when Hash
        setter = proc do |val|
          val = value.merge val if Hash === val
          set option, val, true
        end
      end

      define_singleton("#{option}=", setter) if setter
      define_singleton(option, getter) if getter
      define_singleton("#{option}?", "!!#{option}") unless method_defined? "#{option}?"
      self
    end
  end

  # ...
end
```

