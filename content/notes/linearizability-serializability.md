---
title: "线性一致性、可串行化以及 SSI"
date: 2019-01-08T22:20:32+08:00
draft: true
---

http://www.bailis.org/blog/linearizability-versus-serializability/

最近终于搞懂了这个玩意。之前先入为主了。

> Combining serializability and linearizability yields strict serializability: transaction behavior is equivalent to some serial execution, and the serial order corresponds to real time. For example, say I begin and commit transaction T1, which writes to item x, and you later begin and commit transaction T2, which reads from x. A database providing strict serializability for these transactions will place T1 before T2 in the serial ordering, and T2 will read T1’s write. A database providing serializability (but not strict serializability) could order T2 before T1.

一些有用的链接：

一致性模型的定义： https://jepsen.io/consistency


### SSI ###

SSI 实现的理论基础： https://en.wikipedia.org/wiki/Precedence_graph

SSI 解决什么问题：write skew。
crdb 的这篇文章讲的很清楚： [what-write-skew-looks-like](https://www.cockroachlabs.com/blog/what-write-skew-looks-like/)

crdb ssi 的实现要点：

- [x] WR 解决： READ 相关 key 时，判断是否有 write indent，如果是正在进行的 txn，则需要根据 txn 的优先级做 abort。
- [ ] RW 解决：

