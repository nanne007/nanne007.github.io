---
title: 尾递归和延续传递
date: 2013-11-23T00:22:07+08:00
---

> 本文源自于 [Daniel P.Friedman](http://www.cs.indiana.edu/~dfried/) 的一篇讲稿 - [编程语言研究在程序员教育中所扮演的角色](http://www.cs.indiana.edu/~dfried/dfried/dfried/mex.pdf) 。

上一周，我和几个大学朋友做了一次简单的 workshop，分享的是[尾递归](http://en.wikipedia.org/wiki/Tail_call)和[延续传递](http://en.wikipedia.org/wiki/Continuation-passing_style)。
## 从递归说起

首先来看看一个递归形式的 `sum` 程序：

``` javascript
  function sum(n) {
    if (0 == n) {
      return 0;
    } else {
      return n + sum(n - 1);
    }
  }
```

等同的尾递归形式如下：

``` javascript
  function sum(n, acc) {
    if (0 == n) {
      return acc;
    } else {
      return sum(n - 1, acc + n);
    }
  }
```

由于 nodejs 没有尾递归优化，当 `n` 超过调用栈的最大值时，
上述两种形式的代码都会产生 `RangeError: Maximum call stack size exceeded` 的错误。

当然，这个求和函数是可以直接使用求和公式计算的，
但，举这个例子的目的不在于此，毕竟很多问题是没有公式可循的。
## 重写递归程序

我们可以用寄存器形式来重写这个递归代码。

``` javascript
{
  var n = null;
  var acc = null;
}

function sum() {
  if (0 == n) {
    return acc;
  } else {
    acc = acc + n;
    n = n - 1;
    sum();
  }
}

{
  n = 100000;
  acc = 0;
  sum();
}
```

代码中添加了两个模拟寄存器的变量 `n` 和 `acc` ，
移除了函数的所有参数，通过（保存）“寄存器”将其传递给下一次的函数调用。
这段代码依旧存在上面的调用栈溢出的问题，但已经越来越接近了。

现在，把这个过程放大，以便有一个更清晰的认识。

``` javascript
{
  var n = null;
  var acc = null;
}

function sum() {
  if (0 == n) {
    return acc;
  } else {
    acc = acc + n;
    n = n - 1;
    return sum;
  }
}

{
  n = 100000;
  acc = 0;
  sum();
}
```

函数 `sum` 在运行一次后挂起，再次调用时，从上次的“寄存器状态”重启运行。

认识到这一点之后，重新思考整个运行过程，就可以构造出下面的代码：

``` javascript
{
  var n = null;
  var acc = null;
}

function sum() {
  if (0 == n) {
    return false;
  } else {
    acc = acc + n;
    n = n - 1;
    return sum;
  }
}

function run() {
  while (sum()) { /*do no operation*/ }
  return acc;
}

{
  n = 100000;
  acc = 0;
  run();
}
```

至此，完成了尾递归到迭代的转换（彻头彻尾的循环代码）。
## 接下来...

但接下来还能够做些什么呢？
下面我们引入另一个寄存器 `action` ，消除函数中的 `return` 。

``` javascript
{
  var n = null;
  var acc = null;
  var action = null;
}

function sum() {
  if (0 == n) {
    action = false;
  } else {
    acc = acc + n;
    n = n - 1;
    action = sum;
  }
}

function run() {
  while (action) { /*perform the action*/  action(); }
  return acc;
}

{
  n = 100000;
  acc = 0;
  action = sum;
  run();
}
```

在每一个返回点，将 `sum` 返回的值写入寄存器 `action` ，这样，函数既不依赖传入的参数，也不依赖返回的值。
## 一个小结

现在，已经完成了尾递归到循环的转换。但这种转换本不应该由语言的使用者去处理。
选择一个可以处理此类问题的编程语言吧，不要让让语言创造者的错误成为你的噩梦。
## 延续传递风格（Continuation Passing Style）

对于大多数的递归程序，把他们转换成尾递归形式并不是那么容易。
但这不是问题，接下来的部分会展示一个通用的方法，而不依赖宿主语言的特有功能。
据此方法得到的尾递归形式的代码具有 CPS(Continuation Passing Style) 性质。
### CPS 形式的 `sum`

再来看看最初的这个 sum 函数：

``` javascript
function sum(n) {
  if (0 == n) {
    return 0;
  } else {
    return n + sum(n - 1);
  }
}
```

我们要做的第一件事情就是给每个函数额外添加一个参数，这个参数代表了函数返回后要做的事情。

``` javascript
function identity(income) { return income; }

function sum(n, cont) {
  cur = (0 == n) ? 0 : (n + sum(n - 1, cont));
  return cont(cur);
}

sum(100000, identity);
```

把 `cont` 的计算向 if 语句推进，得到：

``` javascript
function identity(income) { return income; }

function sum(n, cont) {
  if (0 == n) {
    return cont(0);
  } else {
    return cont(n + sum(n - 1, cont));
  }
}

sum(100000, identity);
```

继续推进，则需要处理最后这个包含了递归调用的表达式 `sum(n - 1, cont)` ：

``` javascript
function identity(income) { return income; }

function sum(n, cont) {
  if (0 == n) {
    return cont(0);
  } else {
    return sum(n - 1, function(acc) { return cont(n + acc); });
  }
}
```

通过创建一个延续（continuation）来负责接下来的事情。
（这里，接下来的事情就是：将 `n` 累加到 `acc` ，然后调用外层的延续）
现在，这个递归函数已经被转换成 **接近尾递归** 的调用。
（之所以称为接近尾递归，是因为匿名函数中包含了自由变量，引用了当前调用栈的 `n` 和 `cont` ）

接下来要做的事情就是对自由变量的解引用。
我们知道，本例中，需要解引用的是匿名函数 `function(acc) { cont(n + acc) }` 中的变量 `n` 和 `cont` 。
在 javascript 中，可以用下列的方式实现
（[let binding](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let) 还有待被接受 ）：

``` javascript
(function(n, cont) {
  return function(acc) { return cont(n + acc);
})(n, cont);
```

从而得到：

``` javascript
function identity(income) { return income; }

function sum(n, cont) {
  if (0 == n) {
    return cont(0);
  } else {
    return sum(n - 1, (function(n, cont) {
      return function(acc) { return cont(n + acc); };
    })(n, cont));
  }
}
```

这是一个完整的具有延续传递风格的尾递归程序。
### CPS 形式到寄存器形式的转换

在没有尾递归优化的编程语言（如 C、Java、Ruby）中，上述的的代码依旧会产生调用栈溢出的错误。
不过，仿照文章前半部分，可以很轻易的把它变成迭代形式。

``` javascript
{
  var n = null;
  var acc = null;
  var action = null;
  var cont = null;
}

function identity() { action = false; }

function sum() {
  if (0 == n) {
    acc = 0;
    action = cont;
  } else {
    cont = (function(n, cont) {
      return function() {
        acc = n + acc;
        action = cont;
      };
    })(n, cont);
    n = n - 1;
    action = sum;
  }
}

function run() {
  while (action) { action(); }
  return acc;
}

{
  n = 100000;
  acc = 0;
  cont = identity;
  action = sum;
  run();
}
```

