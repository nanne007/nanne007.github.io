---
title: "重读 DDIA isolation 部分"
date: 2019-08-23T16:57:58+08:00
---

事务隔离要解决以下几类问题：

-读： dirty read, read skew
-写： dirty write, lost update, write skew


#### dirty read (脏读)

在一个 tx 中，读到其他 tx 还没有 commit 的 writes。


RC, SI, RR, Serializable

#### dirty write (脏写)

在一个 tx 中，write 覆盖了其他 tx 中尚未 commit 的 writes。

(加 row-level 的lock，commit/rollback 时释放)

RC, SI, RR, Serializable

#### read skew(nonrepeatable read) (读偏斜)

一个 tx 中，连续读两次相同条件的数据，读到的数据不一样。（因为可能被其他 tx commitle 新的 write 数据）

SI, RR, Seriablizable

#### lost update (写丢失)

一个 tx 中，读了一个数据，然后在修改（比如，read count 为10，然后update count = 10 + 1），最后 commit。如果是多个 tx 同时进行这样的操作，最后 count 值可能不正确。

Solution:
- atomic write
- explicit lock: select .... for update


MySQL InnoDB 的 RR 无法检测 lost update。PG, Oracle, SQL Server 的 SI 可以检测到（abort tx）。



#### write skew

一个tx 中，先读了一部分数据，然后基于某种业务规则限制（比如：当前最少有两个人值班），对数据做write（如果有在值班的人小于2，就把自己设置成值班状态）。
如果这时候，有其他 tx 做了一些 write 操作，导致该 tx 之前读的数据不满足业务规则限制了，那么这时候还要commit 要修改的数据，就会出现逻辑错误。

write skew 是 lost update 的泛化。（write skew write 同一行数据，这种情况下，就是 lost update）


Serializable


#### 解决方式

- 2PL: 悲观锁，读写之前都要加锁，读读互不影响，用共享锁，凡是要写，就要升级成排它锁。
- SSI


### SSI

- 在 SI 的基础上，对串行化冲突做检测。
  - 检测过期的 MVCC 版本的读操作。（在读取某部分数据时，有其他 tx 已经修改了这部分数据，但还未提交）
    读取数据的时候，记录其他修改了这部分数据的 txs，当该 tx 要提交的时候，如果自己也有写操作，那么就检查这些 txs 是否有 commit，如果有commit，那当前的 tx 就要回滚。
    （注意，rollback 的条件是，自己在读之后有写操作，且其他 txs 有 commit）
  - 检测，在读过之后，有其他 tx 修改并 commit 了这部分数据的情况。
    记录读某部分数据的 txs，当前的 tx 在commit 的时候，如果有修改这部分数据，那就通知这些 txs 这部分数据已经被我修改了，其他 tx 收到这个通知后，就知道自己可能读到过期数据，然后可以做回滚操作（如果自己有写操作的话）。
