---
title: "TiDB 的 Update 操作"
date: 2018-02-28T22:46:26+08:00
tags:
  - tidb
---

今天突然想起来一个点：当提交过来的 Update 操作中涉及到了索引列，且列的新值与旧值相同，这个时候，不需要更新对应的索引数据。
好奇 TiDB 是否做了这个优化，于是过了一下对应的代码。

涉及的代码包括：

- https://github.com/pingcap/tidb/blob/master/executor/write.go
- https://github.com/pingcap/tidb/blob/master/table/tables/tables.go

大致流程是这样的，

1. 先通过  `SelectPlan` 把需要 `update` 的数据行从 TiKV 中拿出来，将行中需要修改的字段找出，标记为 assigned。（看到这里，我还以为 TiDB 没有做，差点就没忍住去提 Issue 了）
2. 在 `updateRecord` 方法中，又进一步对上述被标记为 assigned 的字段做判定，判断是否真的被 modified 了。规则是：比较新值和旧值，如果不相等，才标记为 modified。
3. 做完了上面这个事情，如果真的存在修改，再对表中的 `OnUpdateNow` 的字段做处理。
4. 准备完数据后，接下来就是把 column 更新后的值写入到底层 Storage 中。这段代码在 `Table.UpdateRecord` 实现中。忽略 binlog 的处理，其实就是更新索引列的数据（删除旧的，增加新的），以及更新主表的数据。



就是不知道 Phoenix 做 Upsert 时，有没有这个优化。如果没有，那问题就大啦！
每次 Upsert，无论数据是不是真的被修改掉，都需要去做 Write 操作。而 Phoenix 的跨行事务垃圾的很，很容易出问题。
