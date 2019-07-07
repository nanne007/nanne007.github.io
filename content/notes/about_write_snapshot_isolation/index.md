---
title: "Paper: A critique of snapshot isolation 笔记"
date: 2018-12-04
tags:
  - database
  - snapshot-isolation
  - serializable
  - paper
description: write-si 的起源
---


### Section 2 ###

- both write to row r
- 时间重叠：si < cj, ci > sj


介绍了 SI 的两种实现方式：
- 基于锁的实现，拿 percolator 举例。
- 无锁的实现。没细看，跳过了。

### Section 3: Serializablility ###

首先说明了 SI 的问题：

- 避免 Write-Write 冲突不足够实现 Serializablility。有 write skew 的问题。
- 同时也不是实现 Serializablility 的必要条件。禁止了一些事实上是 Serializable 的操作。

#### Section 4: Read-Write vs Write-Write ####

从 read-snapshot isolation 出发定义了 write-snapshot isolation：

- txn2 修改了 txn1 读的数据。
- s1 < c2, c1 > c2

得出结论：
- 避免 Read-Write 冲突可以实现 Serializablility。
- 但同样也禁止了一些是 Serializable 的操作。

