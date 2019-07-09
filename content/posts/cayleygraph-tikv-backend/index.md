---
title: "Cayley 的 TiKV 后端"
date: 2019-07-09T19:49:15+08:00
draft: true
---

## 引言 ##

[Cayley](https://github.com/cayleygraph/cayley) 是 Google 出来的几个哥们写的开源图数据库，主要基于 freebase 和 google 的知识图谱。
[TiKV](https://github.com/tikv/tikv) 是 PingCAP 开源的一个分布式 KV 存储。

今天要讲的就是，怎么把 TiKV 适配成 Cayley 的一个存储引擎。 代码在[这里](https://github.com/lerencao/cayley/tree/feature/tikv-backend)


目前基本的 kv 接口实现完成，通过了 kvtest 的所有测试。
但是还有三个问题需要解决：

- [x]  kv 的 get 接口要利用 snapshot 的 batch get 接口实现，不然性能也是问题。
- [ ] quadstore 会为数据生成唯一的ID，对于kv mode 来说，实现是将这个 id 是存在 local storage 里，每次生成的时候，load 出来，然后 +1，再存回去。
这种方式是写入数据的性能瓶颈。需要改改。
- [ ]  TiKV 需要一个新的 gc handler，tidb 的 gc worker 还没有看，但是肯定不能用。需要过下它的代码，看看可以借鉴什么。

