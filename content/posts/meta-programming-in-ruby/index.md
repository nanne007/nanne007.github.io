---
title: "Ruby 元编程基础"
date: 2014-10-16T00:14:07+08:00
---

## 对象模型

_Ruby_ 的 `class` 关键字更像是一个作用域操作符，而不是类型声明。
`class` 的核心任务是将你带到类的上下文中，让你可以在其中定义方法。
### 对象能做的事情

```
Object#instance_variables()
Object#methods()
Object#singleton_methods()
Object#class()
BasicObject#instance_eval()

Objecy#extend()
```
### 只有类能做的事情

```
Module#instance_methods()
Module#method_defined?()

Module#public_instance_methods()
Module#public_method_defined?()
Module#public_class_methods()

Module#protected_instance_methods()
Module#protected_method_defined?()

Module#private_instance_methods()
Module#private_method_defined?()
Module#private_class_methods()

Module#ancestors()

Module#class_eval()

Module#attr_*() # series ...

Class#superclass()

Module#include()
Module#included()

Module#extended()
```
### 调用方法的过程

在 ruby 中调用方法，需要执行以下两个步骤。
#### 1. 方法查找

**向右一步，再向上**：先向右一步，来到方法接受者所在的类，然后沿着类的祖先链向上找到给定的方法。

其中需要注意的是，在类中包含模块时，模块是如何加入到类的祖先链中的。(`Object` 类中就包含了 `Kernel` 模块)
#### 2. 执行方法

执行方法时，需要知道哪个对象充当当前对象(亦即 `self`)。

类/模块 定义中(任何方法定义之外)的 `self` 就是这个类/模块。

**扩展知识：顶级上下文、私有原则**
## 方法
### 动态方法

动态调用方法：利用 `Object#send()` 动态调用方法，也称 **动态派发**
例如：只调用那些名字匹配某种模式的方法

动态定义方法：利用 `Module#define_method()`，为其提供一个 _方法名_ 和一个 _充当方法主体的块_

**扩展知识：符号和字符串之间的转换方法: `String#to_sym(), String#intern(), Symbol#to_s(), Symbol#id2name()`**
### method_missing 方法

进行方法查找时，如果没有找到这个方法，就会在接收者上调用 `method_missing` 方法。
所有无法投递的消息都会在 `method_missing` 里处理。

所有的 `Object` 实例都会有该方法，因为 `method_missing` 是 `BasicObject` 类的一个实例方法，
并且默认的行为是产生 `NoMethodError` 错误。

``` ruby
BasicObject.private_instance_methods.include? :method_missing # => true
```
### 幽灵方法

所以，可以覆写 `method_missing` 方法，编写自己的逻辑，处理那些并不存在的方法，
从调用者角度看，就好像接收者真的存在这些方法似的，但事实上，并没有。
Ruby 把具有这种行为的方法称为 **幽灵方法**
### 动态代理

动态代理就是一个对象，
这个对象可以利用 `method_missing` 捕获幽灵方法，
然后把它们转发给另一个对象（有时也会在转发前包装一些自己的逻辑）。
### 幽灵方法带来的问题
- 幽灵方法，并不是真正的方法。
- 无法在自动生成的文档中找到，也不会出现在 `Object#methods()` 获得的方法列表中，
- 同样，`Object#respond_to?()` 也不会响应幽灵方法。

可以在覆写 `method_missing` 方法的同时，也覆写 `respond_to?` 方法。。
### const_missing 方法

`const_missing` 是 `Module` 的一个实例方法。
当引用一个不存在的常量时，Ruby 会把这个常量名作为一个符号传递给 `const_missing` 方法。

可以在一个给定的 **命名空间** 中定义该方法，处理该命名空间内的常量。
### method_missing 带来的问题
1. 神秘的bug：在 `method_missing` 中，使用超出作用域范围内的变量
   或调用未定义的方法会导致再次调用 `method_missing` 方法，这样的 bug 往往很难被发现。

   所以，仅仅在必要的时候，再使用幽灵方法。
   最好使用迭代的方式开发程序，先使用普通方法实现，确定没问题，再重构到 `method_missing` 方法中。
2. 方法命名冲突：如果一个幽灵方法与对象的某个真实的方法（可能是继承过来的）发生名字冲突，
   那么会首先调用这个真实的方法，幽灵方法就不会起作用。

   可以通过删除这个真实存在但不需要的方法：使用 `Module#undef_method` 或 `Module#remove_method`，注意区别。
   这样的类，叫做 **白板类** （删除了某些从 `Object` 继承过来的方法）。Ruby 1.9 后，由 `BasicObject` 作为根节点，
   继承它的类自动成为白板类。
3. 性能问题：使用幽灵方法相对来说会更慢，因为方法的查找路径要长些。

   可以折中处理：第一次调用幽灵方法，可以动态创建一个方法，这样以后的调用就直接调用这个动态方法。
## 代码块
### 块、闭包

下面的代码解释了 **块(block)** 的基本用法

``` ruby
def my_method(a_arg)
  another_arg = 2
  yield(a_arg, another_arg) if block_given?
end

my_method(1) { |a, b| a + b } # => 3

# block_given? is private instance method of Kernel
Kernel.private_instance_methods.include? :block_given? #=> true
```

当定义一个块时，块会获取当时环境的绑定，它会带着绑定一同进入使用它的方法中，所谓的闭包特性即在于此。
### 作用域

**要点：什么时候切换了作用域，怎样让 _绑定(binding)_ 穿越作用域。**

可以使用 `Kernel#local_variables` 查看当前作用域的局部变量。

``` ruby
Kernel.private_instance_methods.include? :local_variables #=> true
```

Ruby 会在三个地方打开一个新的作用域：
- 类定义，用 `class` 作为标志
- 模块定义，用 `module` 作为标志
- 方法调用，用 `def` 作为标志（定义方法并不真正打开作用域，只有在调用时，才算是）

只要程序切换了作用域， 绑定也会随之改变。那如何让绑定穿越作用域门呢？

我想答案应该是
- 用 `Class.new` 代替 `class`
- 用 `Module#define_method` 代替 `def`
- 闭包

``` ruby
my_var = "success"
MyClass = Class.new do
  puts my_var
  define_method :my_method do
    puts my_var
  end
end
```

这种方法常被称为 **嵌套文法作用域** 或 **扁平化作用域**

**共享作用域** 是 **扁平化作用域** 一个很好的应用。

``` ruby
# shared scope
# instances of Counter share the same variable-counter
class Counter
  counter = 0
  define_method :value { counter }
  define_method :inc { |x| counter += x }
end

c = Counter.new
c.value #=> 0
c.inc 4
c.value #=> 4
d = Counter.new
d.value #=> 4
d.inc 4
d.value #=> 8
c.value #=> 8
```
### BasicObject#instance_eval() 的魔力

`BasicObject#instance_eval()` 方法赋予了对象打破封装的能力。
给 `instance_eval` 提供的块，把运行它的 obj 作为 `self`，由此，块可以访问 obj 的实例变量和私有/公有方法。
这样的块，被称为 _上下文探针_, 因为这些块可以深入到对象中执行代码。

``` ruby
class MyClass
  def initialize; @v = 1; end
  private
  def my_private; p "private method"; end
end

MyClass.new.instance_eval do
  self
  @v #=> 1
  my_private #=> "private method"
end
```

**TODO：`instance_eval` 能做的不止这些，这里需要补充。**
### 可调用对象

块并不是一个对象，但有时候，我们需要存储一个块供以后执行，Ruby 提供了 `Proc` 类。
通过把块传递给 `Proc#new` 方法，来创建一个 `proc`，之后需要使用的时候，用 `Proc#call` 来调用。
这种技术称为 **延迟执行**。

另外，Ruby 还提供了两个内核方法，用来转换块为 `proc`，分别是 `Kernel#lambda()` 以及 `Kernel#proc()`

``` ruby
Kernel.private_instance_methods.include? :lambda #=> true
Kernel.private_instance_methods.include? :proc #=> true
```

`lambda()` 和 `proc()` 的两个区别：
- `proc` 中的 `return` 关键字 是从调用它的作用域中返回
- `proc` 对参数的个数要求不严格，参数多了，舍弃，参数少了，赋予为 `nil`

方法也是可调用的。
- 使用`Object#method()` 方法获得用 `Method` 的对象表示的方法，用`Method#call`调用。
- `Method` 对象只会在它自身所在对象的作用域中执行。
- `Method#to_proc` 将方法转换为 `proc`。
- `Module#define_method` 将块转换为方法。

