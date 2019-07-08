---
title: 关于 Monad
date: 2014-12-24T00:33:18+08:00
tags:
- monad
- scala
---

## Monad ##

### 定义

一个`monad`包含三个东西：
1. 一个类型构造器`M`，它可以从类型`a`构造出类型`M[a]`。
2. 一个`unit`函数，将类型为`a`的值转换成类型`M[a]`，函数签名为： `a => M[a]`。
3. 一个`bind`函数，通过函数`a => M[b]`将一个`M[a]`转换成一个`M[b]`，函数签名：`(M[a], a => M[b]) => M[b]`。概念类似于`flatMap`。

### 另一种构造方式

`flatMap`通常可以用`map`和`flatten`代替。
类似的概念，Monad中的`bind`函数可以用`fmap`和`join`定义。（`fmap`相当于`map`，`join`相当于`flatten`）
- `fmap`，通过函数`a => b`将一个`M[a]`转换成`M[b]`，函数签名：`(M[a], a => b) => M[b]`。
- `join`，将一个`M[M[a]]`转换成`M[a]`，函数签名：`M[M[a]] => M[a]`。

等价的概念：`bind(m, f) = join(fmap(m, f))`。
### Continuation Monad

``` scala
case class M[+A](in: (A => Any) => Any);

def unit[A](x: A) = M { k: (A => Any) => k(x) }

def bind[A, B](m: M[A], f: A => M[B]): M[B] =
  M { k: (B => Any) =>
    m.in { x: A =>
      f(x).in(k)
    }
  }
```

## 练习题 ##


下面练习题来自于 “[Understanding Monads Using Scala](http://blog.tmorris.net/posts/understanding-monads-using-scala-part-1/index.html)”

``` scala
// A typical data type with a single abstract method
case class Inter[A](f: Int => A) {
  // which is a functor
  def map[B](g: A => B): Inter[B] =
    Inter(n => g(f(n)))

  // and a monad (see unital below)
  def flatMap[B](g: A => Inter[B]): Inter[B] =
    Inter(n => g(f(n)).f(n))
}

// unital: A => F[A]
// Implementations for F=Option and F=Inter
object Unitals {
  def unitalOption[A](a: A): Option[A] =
    Some(a)

  def unitalInter[A](a: A): Inter[A] =
    Inter(_ => a)
}

// Exercises
//
// It is recommended to use only map, flatMap and unital* for
// Option or Inter when implementing the exercises below.
// Any other libraries are acceptable (e.g. List functions).
object Sequencing {
  import Unitals._

  // Exercise 1 of 3
  // ===============
  // Implement a function that returns None if the given list
  // contains any None values, otherwise, all the Some values.
  def sequenceOption[A](x: List[Option[A]]): Option[List[A]] =
    error("todo")

    // SOLUTIONx2 (ROT-13)
    /*
    1)
    k.sbyqEvtug(havgnyBcgvba(Avy: Yvfg[N]))((n, o) => n syngZnc (k => o znc (k :: _)))

    2)
    k zngpu {
      pnfr Avy  => havgnyBcgvba(Avy)
      pnfr u::g => u syngZnc (k => frdhraprBcgvba(g) znc (k :: _))
    }
    */

  // Exercise 2 of 3
  // ===============
  // Implement a function that returns an Inter that applies an Int
  // to all the Inter implementations in the List of Inters and returns
  // all the results.
  def sequenceInter[A](x: List[Inter[A]]): Inter[List[A]] =
    error("todo")

    // SOLUTIONx2 (ROT-13)
    /*
    1)
    k.sbyqEvtug(havgnyVagre(Avy: Yvfg[N]))((n, o) => n syngZnc (k => o znc (k :: _)))

    2)
    k zngpu {
      pnfr Avy  => havgnyVagre(Avy)
      pnfr u::g => u syngZnc (k => frdhraprVagre(g) znc (k :: _))
    }
    */

  // Exercise 3 of 3
  // ===============
  // There is repetition in the above exercises.
  // How might we be rid of it?
  // That is for Part 2.

  def main(args: Array[String]) {
    def assertEquals[A](a1: A, a2: A) {
      if(a1 != a2)
        error("Assertion error. Expected: " + a1 + " Actual: " + a2)
    }

    def assertInterEquals[A](a1: Inter[A], a2: Inter[A]) {
      val testInts = List(1, 2, 0, -7, -9, 113, -2048)
      assertEquals(testInts.map(a1.f(_)), testInts.map(a2.f(_)))
    }

    // sequenceOption
    assertEquals(sequenceOption(List(Some(7),
        Some(8), Some(9))), Some(List(7, 8, 9)))
    assertEquals(sequenceOption(List(Some(7), None, Some(9))),
        None)
    assertEquals(sequenceOption(List()),
      Some(List()))

    // sequenceInter
    assertInterEquals(sequenceInter(List()),
      Inter(_ => List()))
    assertInterEquals(sequenceInter(List(Inter(1+),
        Inter(2*))), Inter(n => List(1+n, 2*n)))
  }
}
```
### 实现 sequenceOption

``` scala
def sequenceOption[A](x: List[Option[A]]): Option[List[A]] =
  x.foldRight(unitalOption(List[A]())) { (opt, mem) =>
    opt flatMap (a => mem map (a :: _))
  }
```

为了说明程序运行过程的细节，下面分析给定的三个测试用例：
#### 例子1：`x = List(Some(7), Some(8), Some(9))`

将 `x` 代入到 `sequenceOption` 中，并展开 `foldRight` 得到
（`ops` 指的是传递给 `foldRight` 的匿名函数）：

``` scala
ops(Some(7), ops(Some(8), ops(Some(9), Option(List()))))
```

下面的代换是计算上述表达式的过程：
1. 首先计算最内的子表达式：

   ``` scala
   ops(Some(9), Option(List()))
   // ==>
   Some(9) flatMap { a =>
    Option(List()) map (a :: _)
   }
   // ==>
   Some(9) flatMap { a =>
    Option(a :: Nil)
   }
   // ==>
   Some(Option(9::Nil)).flatten
   // ==>
   Option(List(9))
   ```

   从而有：

   ``` scala
   ops(Some(7), ops(Some(8), ops(Some(9), Option(List()))))
   // ==>
   ops(Some(7), ops(Some(8), Option(List(9))))
   ```
2. 继续求子表达式：

   ``` scala
   ops(Some(8), Option(List(9)))
   //==>
   Some(8) flatMap { a =>
      Option(List(9)) map (a :: _)
   }
   //==>
   Some(8) flatMap { a =>
      Option(a :: List(9))
   }
   //==>
   Some(Option(8::List(9))).flatten
   //==>
   Option(List(8, 9))
   ```

   又有：

   ``` scala
   ops(Some(7), ops(Some(8), Option(List(9))))
   //==>
   ops(Some(7), Option(List(8, 9)))
   ```
3. 最终求得结果：

   ``` scala
   ops(Some(7), Option(List(8, 9)))
   //==>
   Some(7) flatMap { a =>
      Option(List(8, 9)) map (a :: _)
   }
   //==>
   Some(7) flatMap { a =>
      Option(a :: List(8, 9))
   }
   //==>
   Some(Option(7::List(8, 9))).flatten
   //==>
   Option(List(7, 8, 9))
   ```
#### 例子二：`x = List(Some(7), None, Some(9))`

使用同样的方法展开得到：

``` scala
ops(Some(7), ops(None, ops(Some(9), Option(List()))))
```

代换过程如下：

``` scala
ops(Some(7), ops(None, ops(Some(9), Option(List()))))
//==>
ops(Some(7), ops(None, Option(List(9))))

ops(None, Option(List(9)))
//==>
None flatMap { a =>
  Option(List(9)) map (a :: _)
}
//==>
None

ops(Some(7), ops(None, Option(List(9))))
//==>
ops(Some(7), None)

ops(Some(7), None)
//==>
Some(7) flatMap { a =>
  None map (a :: _)
}
//==>
Some(None).flatten //
==>
None
```

这个例子中，一旦遇到了一个 `None` 值，fold 值就变成 `None`，
这导致后续所有的 `fold` 操作都会是 `None`。
#### 例子三：`List()`

对于 `List()` 来说，因为没有元素，`foldRight` 操作返回的是初始值 `Option(List())`，
也即 `Some(List())`，一个包含空元素列表的 `Some`。
### 实现 sequenceInter

``` scala
x.foldRight(unitalInter(List[A]())) { (it, mem) =>
  it.flatMap (a => mem map (a :: _))
}
```
#### 例子一：`x = List()`

展开 `sequenceInter(List())` 得到 `unitalInter(List[A]())`，即：`Inter(_ => List[A]())`。
#### 例子二：`x = List(Inter(1+), Inter(2*))`

`1+`、`2*` 这样的语法是 Scala 的 `postfixOps`，它允许你使用后缀语法。
在这里，可以写成 `List(Inter(1 + _), Inter(2 * _)`。

展开 `sequenceInter(x)` 得到：

``` scala
ops(Inter(1 + _), ops(Inter(2 * _), Inter(_ => List[A]())))
```

下面是其展开过程：

``` scala
ops(Inter(2 * _), Inter(_ => List[A]()))
//==>
Inter(2 * _).flatMap ( a =>
  Inter(_ => List[A]()) map ( a :: _)
)
//==>
Inter(2 * _).flatMap ( a =>
  Inter( _ => List(a))
)
//==>
Inter(n => Inter(_ => (2 * n) :: List[A]()).f(n)) //
//==>
Inter(n => List(2 * n))

ops(Inter(1 + _), ops(Inter(2 * _), Inter(_ => List[A]())))
//==>
ops(Inter(1 + _), Inter(n => List(2 * n)))

ops(Inter(1 + _), Inter(n => List(2 * n)))
//==>
Inter(1 + _) flatMap ( a =>
  Inter(n => List(2 * n)) map (a :: _)
)
//==>
Inter(1 + _) flatMap (a =>
  Inter(n => a :: List(2 * n))
)
//==>
Inter(m => Inter(n => (1 + m) :: List(2 * n)).f(m))
//==>
Inter(m => (1 + m) :: List(2 * m))
//==>
Inter(n => List((1 + n), (2 * n)))
```
### 结论

使用 Moands 时，应该抓住其背后的逻辑，这样才能以不变应万变。
