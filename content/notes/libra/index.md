---
title: "Libra 源码 - Storage"
date: 2019-07-31T23:43:04+08:00
draft: false
---

storage 本身是一个 grpc 服务。

### 接口 ###

module: storage/storage_service

提供的 grpc 接口：

``` protobuf
service Storage {
    // Write APIs.

    // Persist transactions. Called by Execution when either syncing nodes or
    // committing blocks during normal operation.
    rpc SaveTransactions(SaveTransactionsRequest)
    returns (SaveTransactionsResponse);

    // Read APIs.

    // Used to get a piece of data and return the proof of it. If the client
    // knows and trusts a ledger info at version v, it should pass v in as the
    // client_known_version and we will return the latest ledger info together
    // with the proof that it derives from v.
    rpc UpdateToLatestLedger(
    types.UpdateToLatestLedgerRequest)
    returns (types.UpdateToLatestLedgerResponse);

    // When we receive a request from a peer validator asking a list of
    // transactions for state synchronization, this API can be used to serve the
    // request. Note that the peer should specify a ledger version and all proofs
    // in the response will be relative to this given ledger version.
    rpc GetTransactions(GetTransactionsRequest) returns (GetTransactionsResponse);

    rpc GetAccountStateWithProofByStateRoot(
    GetAccountStateWithProofByStateRootRequest)
    returns (GetAccountStateWithProofByStateRootResponse);

    // Returns information needed for Executor to start up.
    rpc GetExecutorStartupInfo(GetExecutorStartupInfoRequest)
    returns (GetExecutorStartupInfoResponse);
}

```

具体实现在 `storage/storage_service/src/lib.rs` 中的 StorageService 类。实现上也是 delegate 到 libradb 对应的方法。

中间对 libradb 做了一层 wrapper，是为了保证 grpc server 关闭之前，先 close 掉 db。

> See these links for more details.

>   https://github.com/pingcap/grpc-rs/issues/227

>   https://github.com/facebook/rocksdb/issues/649


### LibraDB ###

LibraDB 其实比较简单，由两部分组成：

- SchemaDB：对 RocksDB 实例做了一层 schema 封装。让调用者可以直接 insert/query 一个 schema 对象，它处理对象的序列化和反序列化操作。代码在 `storage/schemadb` 中。
  定义 schema 时，libra 提供了 `define_schema` 的 rust macro，主要用来关联 rocksdb 的 cf name，以及 key/value 的 codec。
  主要的schema 定义在 `storage/libradb/schema` 里。
- 基于 SchemaDB 封装的一些操作组件：不用组件负责操作不同域的对象数据。
  - LedgerStore
  - TransactionStore
  - StateStore
  - EventStore
  - SystemStore


### Data Structures ###


accumulator:

```
//! Example:
//! Logical view of a Merkle Accumulator with 5 leaves:
//! ```text
//!            Non-fzn
//!           /       \
//!          /         \
//!         /           \
//!       Fzn2         Non-fzn
//!      /   \           /   \
//!     /     \         /     \
//!    Fzn1    Fzn3  Non-fzn  [Placeholder]
//!   /  \    /  \    /    \
//!  L0  L1  L2  L3 L4   [Placeholder]
//! ```
```

根据 hashvalue 列表构造的一个二叉树。
当有新的 hashalue append 到列表中时，可以动态的更新二叉树中的节点，得到 root hash。


### What's Next ###

- [ ] sparse_merkle
- [ ] state_view
- [ ] 重要的 schema 分析
