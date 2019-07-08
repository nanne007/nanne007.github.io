---
title: 关于 RocksDB 的 sub-compaction
date: 2018-12-15T22:20:32+08:00
---

之前我一直以为 sub-compaction 只会在 l0-> l1 生效。

今天观察线上 tikv 节点的 rocksdb log，发现 l2 -> l3 的  manual compaction，它的 sub compaction 竟然是 2。我配置的 tikv max_sub_compactions 也是 2。

看了 rocksdb(5.8.7) 的代码：

``` cpp
bool Compaction::ShouldFormSubcompactions() const {
  if (immutable_cf_options_.max_subcompactions <= 1 || cfd_ == nullptr) {
    return false;
  }
  if (cfd_->ioptions()->compaction_style == kCompactionStyleLevel) {
    return start_level_ == 0 && output_level_ > 0 && !IsOutputLevelEmpty();
  } else if (cfd_->ioptions()->compaction_style == kCompactionStyleUniversal) {
    return number_levels_ > 1 && output_level_ > 0;
  } else {
    return false;
  }
}
```

上面只提到了 l0 -> ln。

然而官方文档提到： https://github.com/facebook/rocksdb/wiki/Sub-Compaction

manual compaction 也会使用 sub_compaction 。

需要再探索一下。


### 更新结论 ###


rocksdb 在后面的更新中加入了这个功能：如果是 manual compaction ，也会利用 sub_compaction 加快 compact_range。
PR 在 https://github.com/facebook/rocksdb/pull/3549

TiKV 使用的rocksdb 是 PingCAP 自己 fork 过来维护的。
在 v2.0.2 版本的时候， PingCAP 已经把上述的功能引入到它们自己的 RocksDB 中了。
PR 在 https://github.com/pingcap/rust-rocksdb/pull/202

So far so good
