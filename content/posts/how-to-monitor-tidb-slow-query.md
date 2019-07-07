---
title: TiDB 慢 SQL 收集和监控
date: 2018-12-16 21:33:22
tags:
- tidb
- monitor
description: 介绍了一种如何收集并监控 TiDB 慢 SQL 的方法。
---

本文介绍了一种如何收集并监控 TiDB 慢 SQL 的方法。
首先给出了收集的整体流程图。
然后具体说明了每一步涉及的主要功能以及实现方式。
最后给出了几个应该关注的监控点。

### 流程图 ###

![arch](https://user-images.githubusercontent.com/1456096/50014598-5a358380-ffff-11e8-9775-825ac37de17b.png)

TiDB 将慢 SQL 日志输出到文件中；filebeat 收集这些日志，发送到 Kakfa；
然后通过一个解析慢 SQL 的 consumer 程序，将解析后的结构数据，写入到 InfluxDB；
最终用 Grafana 来展示和监控慢 SQL。

### TiDB 慢日志配置 ###

TiDB 有两个选项用来配置慢 SQL 日志：

```toml
# Stores slow query log into separated files.
slow-query-file = ""

# Queries with execution time greater than this value will be logged. (Milliseconds)
slow-threshold = 300
```

也可以直接通过命令行参数 `--log-slow-query** 指定慢SQL 日志文件位置，优先级比配置文件高。

**默认是不配置的，慢 SQL 日志会和其他日志一起输出。**

配置完成之后，慢 SQL 日志会记录到单独的日志文件中，比如 `tidb_slow_query.log`。

### Filebeat 收集 ###

然后，为每个 TiDB Server 起一个 [filebeat](https://github.com/elastic/beats/tree/master/filebeat)，用来收集该 TiDB Server 的慢 SQL 日志。

> filebeat 是用 Golang 编写的一个日志收集工具，和 logstash 相比，小巧方便易用。


这里需要配置 filebeat 的 input 类型为 `log`，路径是前述慢 SQL 日志文件的路径。

``` yaml
filebeat.inputs:
  - type: log
    enabled: true
    fields:
      log_type: slow_query
    paths:
      # change the path to your tidb log path
      - /data/tidb/log/tidb_slow_query*.log

    fields_under_root: true
    multiline.pattern: '^\d{4}/\d{2}/\d{2}'
    multiline.negate: true
    multiline.match: after
    multiline.max_lines: 100
    multiline.timeout: 2s
    tail_files: true
    max_backoff: 5s
```

output 配置成 kafka 的地址（发送到 kafka 是为了进一步解析）:

>  和 logstash 不同，filebeat 没有太多的解析功能。

``` yaml
output.kafka:
  hosts:
    # change the host:ip to your kafka address
    - 172.16.1.1:9092

  topics:
    # change the topic name as you wish
    - topic: tidb.logs.slow_query
      when:
        equals:
          log_type: slow_query

  required_acks: 1
  compression: gzip
  worker: 2
  client_id: tidb_log_beats
```

### 日志格式解析 ###

从 kafka 里消费慢 SQL 日志数据，利用正则表达式：
``` regexp
(\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}\.\d{3}) .+? \[SLOW_QUERY\] (?:\[INTERNAL\] )?cost_time:(\S+)(.+?)succ:(\S+?) con:(\d+?) user:(.*?)(?:@(.*?))? txn_start_ts:(\d+?) database:(\S*?) (?:table_ids:\[(.*?)\],)?(?:index_ids:\[(.*?)\],)?sql:((?:.|\s)+)
```
解析出里面的字段：

```json
[
  "cost_time",
  "succ",
  "con",
  "remote_user",
  "remote_ip",
  "txn_start_ts",
  "database",
  "table_ids",
  "index_ids",
  "sql",

  "process_time",
  "wait_time",
  "backoff_time",
  "request_count",
  "total_keys",
  "processed_keys"
]
```

正则表达式是参照 TiDB SLOW\_QUERY 的输出逻辑编写的。具体见：

https://github.com/pingcap/tidb/blob/808eab19115d81a2940bca87e287dd7e58218c75/executor/adapter.go#L390


> 这里需要注意的是，不同版本的 TiDB SLOW\_QUERY 的输出格式不一样。
> 目前我司线上 TiDB 版本从 1.x => 2.0 => 2.1，甚至有多个版本共存，所以需要根据版本对正则表达式做不同处理。
> 上述正则表达式可以解析 2.0 和 2.1 版本的 SLOW\_QUERY。

> 更新：
> 3.0 以后使用的是类似 mysql 的慢日志格式，上述正则已经不再适用。

拿到这些数据后，需要将以下 Golang 的时间字符串（类似 `308.334206ms`）转换成 double 值，以便于 grafana 做监控。

```
cost_time
process_time
wait_time
backoff_time
```

最后，将这些数据以 influxdb 要求的方式组织起来，写入到 influxdb 中。

以上逻辑搞定后，再通过 flink 任务（或者其他 kafka consumer 形式）接入到 kafka 里。
整个收集过程基本完毕。

### Grafana 监控 ###

数据入到 InfluxDB 后，就可以在 Grafana 中建立统计指标，做实时监控。

这里主要推荐两个监控：

- Full Scan 的 SQL 查询。 可以通过 `index_ids` 来判断是否是全表扫描，`index_ids` 为空，说明没有走索引查询。再结合主键是否 row handle，就可以判断查询是否是全表扫描，并迅速定位 user 和 ip。
- 某个用户，某张表在过去10分钟的慢 SQL 统计。


通过慢 SQL 监控，可以实时的感知当前 TiDB 集群的服务状态以及 PD 调度速率对用户的影响。

比如说，TiKV 掉线，region leader 发生重新选举，对查询的影响比较大，耗时可到几十秒，持续时间与 TiKV 节点上的leader 个数有关。

再比如，PD Leader 的主动切换基本不影响查询，而 TiKV evict-leader 速度过快会影响查询。

如此等等。


### 总结 ###

这套 TiDB 慢 SQL 监控方式，将 filebeat，以及 kafka，influxdb 这些基本组件组合在一起，各个组件分工明细，结构清晰，目前尚未出现问题。

如果使用 logstash，可以省掉 kafka 这一步骤，直接在 logstash 中解析出结构数据。但基本的处理流程应该是不变的。


最后，希望本文能够帮助到正在使用 TiDB 的你。


