---
title: "Tikv 的参数调优"
date: 2018-04-08T22:51:24+08:00
---

### Manual Compaction ###

TiKV 有一个后台任务定期去对 region 做Compact，每次 100个 region。见：`store.on_comact_check_tick`。

落到  RocksDB里面就是很多个 CompactRange 操作。见： `compact.rs`

如果业务场景是频繁的写更新，一次 tick 就会产生多个 compact_range 操作。
而 RocksDB 在 CompactRange 时会做一次 flush。
如果某次 compact range 的次数过多，会引起 level-0 的sst 来不急 compaction，引起 write_stall。

参考：

- http://kernelmaker.github.io/Rocksdb_Study_4
- http://kernelmaker.github.io/Rocksdb_Study_5
- http://alexstocks.github.io/html/rocksdb.html


### 系统参数 ###

fs： dirty_background_ratio 和 dirty_ratio

参考：

- https://www.jianshu.com/p/027f681f59e6

### bytes-per-sync ###

https://github.com/tikv/tikv/issues/2675
