---
title: "accumulate 实现上的坑"
date: 2014-09-10T21:01:14+08:00
---

今天看 SICP 时，遇到了 `accumulate` 的实现。（类似的还有，Ruby中的 `inject`，Scala中的 `fold`）

``` racket
(define (accumulate op initial sequence)
  (if (null? sequence)
      initial
      (op (car sequence)
          (accumulate op initial (cdr sequence)))))
```

自我聪明写出了下面这样的代码：

``` racket
(define (accumulate op initial sequence)
  (if (null? sequence)
      initial
      (accumulate op (op (car sequence) initial) (cdr sequence))))
```

做习题用到这个函数时，才察觉出和原文实现的差别。

原文的实现，是从右到左的，而我的则相反。比如说，对于 `(accumulate - 0 '(5 4 3 2 1))`，从右到左是这样的：

``` racket
(- 5 (- 4 (- 3 (- 2 (- 1 0))))) ; => (5-(4-(3-(2-(1-0)))))
```

而从左到右则大不相同：

``` racket
(- (- (- (- (- 0 5) 4) 3) 2) 1) ; => (0-5-4-3-2-1)
```

这也是为什么，从左到右时，`initial` 一般是 `op` 的第一个参数，而从右到左时，却是第二个。

想到这里的时候，脑子里突然冒出了 [reduce-fold-or-scan-left-right - stackoverflow](http://stackoverflow.com/questions/17408880/reduce-fold-or-scan-left-right)，这是在了解 Scala 时碰到的。当时还以为自己懂了...

要想保证从左到右和从右到左进行 `fold` 操作都得到一样的结果，需要满足下面这两个条件：

- op满足结合律，所谓的monoid。

  ``` racket
  (= (op (op a b) c) (op a (op b c)))
  ```
- 初始值时，op要满足交换律。

  ``` racket
  (= (op init a) (op a init))
  ```

