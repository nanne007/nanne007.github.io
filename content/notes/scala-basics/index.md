---
title: Scala 基础笔记
date: 2013-08-10T00:25:23+08:00
tags:
- scala
---

### 控制结构和函数

- 赋值动作本身是没有值的。

  ``` scala
  var a = 1
  scala> val nn = (a+=1)
  nn: Unit = ()
  ```
- scala 支持函数。只要函数不是递归的，就不需要指定返回类型。
- 异常中 throw 有特殊的类型 `Nothing`。

### 类、对象以及特质

- 下面代码中，`_` 是什么意思？

  ``` scala
  class Person {
      var name: String = _
  }
  ```
- 类型投影？
- 在内嵌类中访问外部类？
- 定义只读属性时,
  - `val attr`
  - `private var privateAttr` + `def attar = privateAttr`

  选择哪个？其他更好的方案？

- 类的伴生对象，可以访问类的私有成员。
- 类和其伴生对象可以相互访问私有特性？
- 单例对象和伴生对象的关系？
- 扩展类
  - 使用 `extends` 关键字扩展类。
  - 可以将类、字段、方法声明为 `final`，以确保其不会被重写。注：java 中的 `final` 意思与 `val` 类似，表示不可变。
- 重写方法
  - 使用 `override` 修饰符来重写一个 **非抽象方法** （那抽象方法？）。
  - 使用 `super` 关键字来调用超类的方法。
- 类型检查和转换
  - `o.isInstanceOf[C]`
  - `o.asInstanceOf[C]`
  - `classOf[C]`

  通常，模式匹配是更好的选择。[?]
- 受保护的字段和方法
  - 使用 `protected` 声明声明字段或方法，这样，就只能被子类访问。
  - 类所属的包不能访问 `protected` 成员。
  - `protect[this]` 将访问权限限制在当前对象，就像 `private[this]`。
- 超类的构造
  - 只有主构造器才能调用超类构造器，像这样:

   ```
   class Employee(name: String, age: Int, val salary: Double) extends Person(name, age)
   ```

   类似的 java 代码如下：

   ```
    public class Employee extends Person {
        private final double salary;
        public Employee(String name, int age, double salary){
            super(name, age);
            this.salary = salary;
        }
    }
   ```
  - Scala 可以扩展 Java 类，但其主构造器必须调用超类的某一构造器方法。
- 重写字段
  - scala 字段由一个私有字段和取值器/改值器方法构成。
  - 一个val（或一个不带参数的 def）可以被另一个同名的 val 字段重写。更多情况如下：
    - def 只能重写另一个 def
    - val 只能重写另一个 val 或不带参数的 def
    - var 只能重写另一个抽象的 var
- 匿名子类 [?]
  - 通过**包含带有定义或重写的代码块**来创建。比如：

    ```
    class Person(val name:String)
    val alien = new Person("Fred") {
        def greeting = "Hello, my name is Fred!"
    }
    ```

    该匿名子类的类型为 `Person{def greeting: String}`。注：技术上，上述代码会创建一个 结构类型[?] 的对象。
- 抽象类
  - 使用 `abstract` 关键字来标记不能被实例化的类。
  - 抽象方法不需要使用 `abstract` ，只需省去其方法体。
  - 如果一个类存在抽象方法，此类必须声明为 `abstract` 。
  - 子类中，重写超类的抽象方法 不需要使用 `oberride` 关键字。
- 抽象字段
  - 类可以拥有抽象字段（没有初始值，即表示其是抽象字段）。
  - 生成的 java 类不带该字段。
  - 具体的子类必须重写抽象字段，无需使用 `override` 。
  - 也可用匿名子类来定制。
- 构造顺序和提前定义 [?]
- Scala 继承层级
  - `Boolean Byte Char Short Int Long Float Double Unit` 都扩展自 `AnyVal` 。
  - 所有其他类都扩展自 `AnyRef`。
  - `AnyVal` 和 `AnyRef` 扩展自 `Any` 。`Any` 是整个继承体系的根节点。
  - 所有 Scala 类都实现了 `ScalaObject` 标记接口。
  - `Nothing` 和 `Null` 类型都直接扩展自 `Any` 。**`Nothing` 是所有类型的子类型，没有实例；`Null` 是所有非值类型的子类型，其只有一个实例，即 `null`（所以，可以将 `null` 赋值给任何引用，但不能赋值给值类型）。**
- 对象相等性
  - `AnyRef` 的 `eq` 方法检查应用是否指向同一个对象。
  - `AnyRef` 的 `euqals` 方法直接调用 `eq` 方法。
  - 当实现一个类时，应该考虑是否重写 `equals` 方法。
  - 若重写了 `equals` 方法，还应重写 `hashCode` 方法。

  > 确保重写的 `equals` 方法参数类型为 `Any` 。重写 `equals` 和 `hashCode` 并不是一种义务。

- 特质与类的两个区别点
  1. 特质不能有类参数
  2. 特质中，对 super 的处理遵循 linearization

## 协变、逆变

-   `Array` 不是协变的，所以

  ```
  val a: Array[SubClass] = Array(...)
  val b: Array[SuperClass] = a
  ```


### Array 的几个关键点

-   Scala 中的 `Array` 和 Java 中的没什么不同，只不过 Scala 给 `Array` 赋予了两个隐式转换，
  一个是到 `ArrayOps`，一个是到 `WrappedArray`
-   不能简单的使用 `==` 来判断两个 array 是否相等，可以用 `array.sameElements(other)` 方法

## List 概念、操作

-   List 是不可变的，且是递归形式的。
-   通过 `Nil` 和 操作符 `::` 来构造。例如：

  ```
  1 :: 2 :: 3 :: 4 :: Nil
  ```
-   三个最重要的方法：`head, tail, isEmpty`
-   List 上的模式匹配：
  -   Nil
  -   x :: xs
  -   List(p1, p2, ..., pn)
-   `length, last, init, take, drop` 以及 `list(n)`
-   `list1 ++ list2`, `list updated (n, x)`
- `filter, takeWhile, filterNot, dropWhile, partition, span`

## Codeing Tips

- Use expressions not statements. In Scala, a lot of code can be written as small methods of one expression, for elegance, and code maintenance.
- Prefer Immutability. Creating immutable classes drastically reduces the number of potential runtime issues.it’s safest to stay immutable, when in doubt.

  _Difference between an immutable object and an immutable reference_, the immutability of the reference doesn’t affect whether the object referred to is immutable.

- HASHCODE AND EQUALS SHOULD ALWAYS BE PAIRED. If x == y, then x.## == y.##(`##` is an alias of `hashCode`)
  - 如果两个对象相等，那么它们应该有相同的 `hashCode`。
  - 在对象的生命周期内，`hashCode` 不应该有所改变。
  - 当发送对象到另外一个 JVM 时，对象相等性的计算应该使用哪些对两个JVM都可用的属性（绝大部分情况下，也就是对象内部的属性）。
- Use `None` instead of `null` .
- Override methods with named parameters and default values.

