---
title: "Cayley 的 TiKV 后端"
date: 2019-07-09T19:49:15+08:00
draft: true
---

## 引言 ##

[Cayley](https://github.com/cayleygraph/cayley) 是 Google 出来的几个哥们写的开源图数据库，主要基于 freebase 和 google 的知识图谱。
[TiKV](https://github.com/tikv/tikv) 是 PingCAP 开源的一个分布式 KV 存储。

今天要讲的就是，怎么把 TiKV 适配成 Cayley 的一个存储引擎。

