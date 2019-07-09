---
title: Ruby 中的方法
date: 2013-04-23T21:12:11+08:00
description: 近距离了解 Ruby 方法
tags:
- ruby
---


## 对象化的方法（objectified method）

一直以来，关于对象方法，我的了解仅止步于使用 `Object#methods`、`Class#instance_methods` 这样的方式去认识对象和类。
而没有进一步去查看对象方法到底是什么，有什么，怎么用。

其实在 Ruby 中，有具体的类来表示 **对象方法**。一个是 `Method`，一个是 `UnboundMethod`。

## Method 类

`Method` 实例实际上就是绑定在一个特定对象的 proc，只不过这个 proc 的实体由方法定义，所以叫做 **对象化的方法**。

用 `Object#method` 方法可以得到这样的一个实例。

``` ruby
class A
  def initialize(a)
    @a = a
  end

  def a(b)
    '#{@a} with #{b} in method A#a'
  end
end

meth = A.new(1).method(:a) #=> #<Method: A#a>
```

得到 `Method` 实例后，可以使用 `Method#call` 或者 `Method#[]` 来调用，效果和在对象上直接调用是一样的。

``` ruby
meth.call(2)   #=> "1 with 2 in method A#a"
meth[2]       #=> "1 with 2 in method A#a"
A.new(1).a(2)  #=> "1 with 2 in method A#a"
```

除此之外，也可以
- 用 `Method#name` 得到方法名
- 用 `Method#receiver` 得到方法绑定到的对象
- 用 `Method#owner` 得到方法所属的类或模块
- 用`Method#to_proc` 将其转换成对应的 lambda

``` ruby
obj = A.new(1) #=> #<A:0x007f88d40fdac0>
meth = obj.method(:a) #=> #<Method: A#a>

meth.name #=> :a
meth.receiver #=> #<A:0x007f88d40fdac0>
meth.to_proc #=> #<Proc:0x007f88d5024df0 (lambda)>
```

最后，用 `unbind` 将 `Method` 实例从对象上解绑，得到 `UnboundMethod` 实例。

## UnboundMethod 类

`UnboundMethod` 实例本质上是一个没有执行上下文的 proc，
而且只有在重新绑定到所属类的某个对象，成为 `Method` 实例后，才能被调用。

可以通过 (1)解绑 `Method` 实例，(2) 调用 `Module#instance_method` 这两种方式得到。

``` ruby
unbound_meth = A.instance_method(:a) #=> <UnboundMethod: A#a>
unbound_meth = meth.unbind #=> #<UnboundMethod: A#a>

new_obj = A.new(10)

new_obj.kind_of?(unbound_meth.owner) #=> true

# bind operation will fail, if last sentence return false
meth = unbound_meth.bind(new_obj)
meth.call(2) #=> "10 with 2 in method A#a"
```
## 怎么使用

Sinatra 中，有这样一个应用场景。

``` ruby
module Base
def self.generate_method(method_name, &block)
  define_method(method_name, &block)
  method = instance_method method_name
  remove_method method_name
  method
end
```

Sinatra 利用这样的方式先保存 _before/after filter_ ，之后将 `filter`s 动态绑定到具体的请求上，在其上下文中执行。

