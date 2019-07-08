---
title: Loop and a half 问题
date: 2013-03-29T00:31:12+08:00
---

大部分的循环形式可以总结如下：

```
A: S;
    if B then goto Z fi;
    T;
    goto A;
Z: // do other stuff
```

当 `S` 为空时，可以写成 `while` 循环：`until B do T;`；
当 `T` 为空时，可以写成 `do-while` 循环：`repeat S; until B;`。

但是，当 `S` 和 `T` 都不为空时，无论是写成 `while` 还是 `do-while` 总会有多余的一部分在循环之外执行：

```
S;
until B do
  begin
    T;
    S;
  end
```

```
inversedT;
repeat
  T;
  S;
until B;
```

这就是所谓的 **loop and a half** 问题。（相信大家在编程过程中都有遇到过）

诸如：`repeat begin S; when B exit; T; end` 中这样的 _loop/break_ 控制语句被添加到现代编程语言中，以解决 消除 `goto` 所带来的不便。

