---
title: 勾股数与 Euclid's formula
date: 2014-07-10
---

勾股定理想必大家都知道，勾股数就是勾股定理中的那三个数组成的数对，即 a<sup>2</sup> + b<sup>2</sup> = c<sup>2</sup> 中的 (a, b, c)。

那 Euclid's formula 是什么呢？维基上是这么解释的：给定两个正整数 m 和 n，其中 m > n，可以用 Euclid's formula 来生成勾股数。

公式如下：

a = m<sup>2</sup> - n<sup>2</sup>，b = 2mn，c = m<sup>2</sup> + n<sup>2</sup>

请注意，**这个公式并不能生成所有的勾股数**，但是使用另外一种方法就可以。

### 生成所有勾股数
1. 首先说明一下素勾股数的概念：如果勾股数 a, b, c 三者之间互质（最大公因数为1），那么就称之为素勾股数。
2. 而所有的勾股数都可以由素勾股数乘以一个正整数 k 得到：k(a, b, c)，其中，(a, b, c) 是素勾股数。

那现在的问题就变成如何生成素勾股数了。方法还是 Euclid's formula 。

> 所有的素勾股数都可以由一对互质的整数 m 和 n（其中 m > n，且其中一个是偶数）通过 Euclid' formula 生成。

通过 Euclid' formula 就可以生成所有的素勾股数。

### 总结

Euclid' formula 虽然可以生成所有的素勾股数，但无法生成所有的勾股数。必须通过素勾股数外加一个乘因子得到，即：

a = k(m<sup>2</sup> - n<sup>2</sup>)，b = k(2mn)，c = k(m<sup>2</sup> + n<sup>2</sup>), 其中，m 和 n 互质，m > n，且其中一个是偶数。

### 更新1 ###

刚刚发现另一种生成所有的素勾股数的方法：[Tree of primitive Pythagorean triples](https://en.wikipedia.org/wiki/Tree_of_primitive_Pythagorean_triples)。

太优雅了！
