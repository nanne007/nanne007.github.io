---
title: "Libra Move 采坑记"
date: 2019-09-25T20:43:04+08:00
draft: false
---

最近开始上手写 move 智能合约，拿石头剪刀布当作业。这里记录下遇到的问题。

首先是 move ir 的编译问题。

libra client 里面有 dev 命令，可以帮助 编译，发布和执行 move 的代码。
但是这个流程，手动操作起来还是不够方便，尤其是在刚刚上手写的时候，经常会遇到各种各样的问题。
libra 测试代码中提供了一个简单的 dsl，来编译执行 move 的代码，具体是在 language/functional_tests 中。

另外就是 libra 的小坑了：

我在 functional test 中写好 module 和 script 后，正常流程的测试也搞定了，
然后放到 libra dev 环境中跑，
用 `dev execute` 执行， tx commit 后，账户的数据却没有发生变化，也不知道问题出在哪里。

后来看了下源码，发现 libra 没有把 tx 执行后的详细结果存下来（只存了一个概要结果，但是概要结果中没有 tx 执行的结果 code）。
libra 源码中加了一行日志，打印 tx 的执行结果，然后，再重新执行合约，日志中报错 OUT_OF_GAS。
dev 命令中没有显式指定 gas 费，那应该就是 client 中写死了一个默认值。
查看源码，发现是 `const MAX_GAS_AMOUNT: u64 = 140_000;` 这么多。
我用 `functional_tests` 执行了一遍合约，得到的结果，gas 费是 *170_0000* 还多，确实超了这个默认值。
最后没办法，把代码中默认的 gas 费往上加了加，合约执行成功。

如果你想本地玩玩，可以试试我的 [lerencao/RockScissorsPaper.mvir](https://github.com/lerencao/RockScissorsPaper.mvir.git)
