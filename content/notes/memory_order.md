---
title: mermory barrier 阅读笔记
date: 2019-03-14
tags:
  - cache coherence
  - memory order
description: 关于缓存一致性和内存屏障的阅读记录
---

https://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html 的阅读笔记。

memory barrier 保证 barrier 两边的指令执行结果可以以程序顺序的方式被其他 CPU 观察到。


store barrier:

> 'sfence' instruction on x86, waits for all store instructions prior to the barrier to be written from the store buffer to the L1 cache for the CPU on which it is issued.
> All previous updates to memory that happened before the barrier are now visible.

visible 只是说 数据写到 cache 了，但还不能保证其他 cpu 的 cache 也是最新的。

load barrier:

> “lfence” instruction on x86, ensures all load instructions after the barrier to happen after the barrier
> and then wait on the load buffer to drain for the issuing CPU.

保证 barrier 之后的 load 指令一定发生在 barrier 之后，不能被重排到 barrier 之前。
并 flush load buffer 中的指令，去获取数据（from other cpu cache or memory）。
也就是说 barrier 之后的 load 指令要在等待之前 load 指令都获取到最新值之后才能执行。


### volatile in Java ###

All writes that occur before a volatile store are visible by any other threads with the predicate that the other threads load this new store.

在一个线程里更新 volatile 字段可以保证，当其他线程看见这个 volatile 字段的更新值后，也能看见这个线程在之前做的更新操作。

### 为什么要有 memory order ###


**加了 Store Buffer 之后，存在 store buffer 中的新值无法被其他 CPU 感知到。**
**另外，invalidate queue 中的 message 被处理完后，才能正常读到新值。**

