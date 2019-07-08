---
title: 质数生成器在 Ruby 标准库中的实现
date: 2013-12-09T00:19:12+08:00
tags:
- ruby
---

如果你在 [Project Euler](http://projecteuler.net) 上做过题，你一定碰见过很多和质数相关的题目。
1. 判断一个数是否是质数，
2. 第XX个质数，
3. 前XX个数中的所有质数，
4. 质因数分解，

最近，遇到了 [Euler's Totient function](http://mathworld.wolfram.com/TotientFunction.html), 和质因数分解有些关系，心血来潮看了下 Ruby prime gem 的代码。
## 质因数分解

质因数分解的代码异常简练，使用的是最简单直接的[试除法](http://en.wikipedia.org/wiki/Trial_division)，借助了质数生成器。
## 生成器

质因数分解默认的生成器是 `Prime::Generator23`：正如名字所暗含的一样，`Generator23` 生成了所有不被2和3整除的数，代码的实现非常灵巧。

> 我也是使用这种方法实现素数判断的，但代码远远不如它精简。好的代码可以提高 Programmer 的觉悟。

但，一个数不被 2 和 3 整除不代表它就是素数，`Generator23` 只是一个伪素数生成器。
它的父类是 `PseudoPrimeGenerator`，`PseudoPrimeGenerator` 包含 `Enumerable`，实现了 `each` 方法；同时，作为父类，又提供 `succ`、`next`、`rewind` 抽象方法（Ruby 中没有抽象方法，这里只是借助其概念）。

除了 `Generator23`，`PseudoPrimeGenerator` 还有两个子类，
- EratosthenesGenerator
- TrialDivisionGenerator
### 生成器 - EratosthenesGenerator

`compute_primes` 是这个生成器的核心。
该实现在基本的 Sieve of Eratosthenes 算法上做了两点改进：
- 偶数直接筛掉。
- 只计算到不大于 `sqrt(n).floor` 的那个质数。

`(-(segment_min + 1 + sieving_primes[i]) / 2) % sieving_primes[i]` 这行代码是画龙点睛之笔。不过，代码可以更加函数化，比如说：

``` ruby
segment.each do |prime|
  @primes.push prime unless prime.nil?
end

# ===>

@primes.concat segment.reject {|p| p.nil? }
```
## Prime 中的元编程

Ruby 中，元编程无孔不入：

``` ruby
class Prime
  include Enumerable
  @the_instance = Prime.new

  #...

  class << self
    extend Forwardable
    include Enumerable
    # Returns the default instance of Prime.
    def instance; @the_instance end

    def method_added(method)
      (class<< self;self;end).def_delegator :instance, method
    end
  end
end
```

因为 `method_add` 方法，才会 `Prime.each` 这样的调用。

