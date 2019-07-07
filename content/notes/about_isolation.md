---
title: "Paper: A critique of ANSI SQL isolation levels 笔记"
date: 2018-09-10
tags:
  - database
  - isolation-level
---


### P1(Dirty Read): 脏读 ###

```
w1[x] ... r2[x] ...
```

- txn1 修改了一行数据。（未 commit 之前）
- txn2 读到修改的值。（未commit 之前）



### P2(Fuzz or Non-repeatable Read): 不可重读 ###

```
r1[x] ... w2[x] ... c2 ... r1[x] ... c1
```

- txn1 读了一个数据，
- txn2 修改这个数据，并 commit
- txn1 重新读这个数据，读到新的值。



### P3(Phantom): 幻读 ###

```
r1[P] ... w2[y in P] ... c2 ... r1[P] ... c1
```

- txn1 读了一段满足某个 condition 的数据，
- txn2 修改了一条数据，使之满足或者不满足这个 condition（比如说删掉了一条满足 condition 的数据，新增了一条满足 condition 数据），并 commit。
- txn1 重新读满足这个 condition 的数据，返回数据变化。

P2 可以看做是 P3 的特殊 case。



### P0(Dirty Write): 脏写 ###

```
w1[x] ... w2[x] ... ((c1 or a1) and (c2 or a2) in any order)
```

- txn1 修改了一条数据。
- txn2 再次修改同一条数据。

txn2 应该 abort 或者 等待 txn1 结束。否则会破坏数据约束。

比如： `w1[x = 10] w2[x = 20] w2[y = 20] c2 w1[y = 10] c1` 会违背 x = y 这个约束。



论文给出：

| Isolation Level  | P0: Dirty Write | P1: Dirty Read | P2: Fuzzy Read | P3: Phantom  |
|:----------------:|:---------------:|:--------------:|:--------------:|:------------:|
| READ UNCOMMITTED | Not Possible    | Possible       | Possible       | Possible     |
| READ COMMITTED   | Not Possible    | Not Possible   | Possible       | Possible     |
| REPEATABLE READ  | Not Possible    | Not Possible   | Not Possible   | Possible     |
| SERIALIZABLE     | Not Possible    | Not Possible   | Not Possible   | Not Possible |


但是还有其他的 anomaly !!!




### P4(Lost Update): ###

```
r1[x] ... r2[x] ... w2[x] ... w1[x] ... c1 ... c2
```

- txn1 读了一条数据。
- txn2 读了并修改了这条数据。
- txn1 修改这条数据，然后 commit。
- txn2 commit。

txn1 做的修改不见了。



### A5A(Read Skew): 读偏斜 ###

```
r1[x]...w2[x]...w2[y]...c2...r1[y]...(c1 or a1)
```

- txn1 读了 x。
- txn2 写了 x 和 y，然后 commit。
- txn1 再读 y，读到新值。

txn 读到的 x，y 不满足某个约束（比如 x==y）。

### A5B(Write Skew): 写偏斜 ###

```
r1[x]...r2[y]...w1[y] ...w2[x]...(c1 and c2)
```

- txn1 读了x。
- txn2 读了 y。
- txn1 写了 y。
- txn2 写了 x。
- 两个 txn 做 commit。

最后 x，y 可能不满足某个约束。比如（x+y=2）


