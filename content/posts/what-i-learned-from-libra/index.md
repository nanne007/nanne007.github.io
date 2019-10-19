---
title: "我从 Libra 学到了什么"
date: 2019-10-18 22:59:26
tags:
  - libra
  - blockchain
  - rust
description: "记录了这两周在 libra 项目上学到的 rust 经验和 blockchain 知识。"
---

## 前言

本文会介绍我在最近一个月里面，对 libra 代码的摸索过程，从中学到了的 Rust 编程经验， 以及 Libra 现在还缺乏的功能，blockchain 开发者能够在 Libra 上做些什么。

## 摸索 Libra 代码

推荐的模块走读顺序：`admission_control => mempool => executor => vm => network => storage => consensus` 。

具体每个模块做什么，这里不展开讲，可以直接参考 Libra 官网。

### actor model

Libra 采用 actor model 编程方式，映射到 Rust 中，actor 是一些跑在 tokio executor 上的 future tasks，

actors 之间利用 rust channels 来传递自定义的 message 来沟通，主要有两种使用方式：

- rpc: actor 在发出去的消息中附带一个 oneshot_sender，然后 wait 对应的 oneshot_receiver，接收消息的 actor 处理完请求后，把响应从 oneshot_sender 发送出去即可。
- direct_send: actor 直接发送消息出去，系统不能确保接收者一定能收到。（假如 channels 满了，后来的消息可能会丢掉，取决于发送端的调用行为）。

_rpc_ 和 _direct_send_ 分别对应 erlang otp 中的 `:call` 操作和 `:cast` 操作。只不过 Rust 里 actor 的执行调度是 tokio 负责的。（tokio 调度器最近被重写了，性能大幅度提升，当然也是借鉴了现有的 goroutine 调度器的实现优化，比如 Golang, Java, Erlang，具体可参考 [tokio scheduler got 10x improvements][tokio-scheduler]。

_network_ 模块是使用 actor model 的典范。 _network_ 实现了 libp2p 的功能，比如加密（noise）、connection 复用（mux）、custom protocol。actor model 体现在两个地方：

- 基础网络层的连接管理，涉及本地 listen，以及 remote peer dial。每个进来、出去的连接都会有一个 peer 直接管理，peer 收到、发出的消息经 main task 路由。

- custom protocol 的实现也有 actor model 的影子。custom protocol 包含有 _rpc_ 和 _direct_send_ 两种操作模式。每个 _rpc_ 操作都会 open 一个新的 substream，在 substream 上和另外一台机器上的某个 actor 构建消息渠道。而 _direct_send_ 的所有消息会复用同个 substream。

  TODO: 思考为什么 rpc 需要每次都新开一个 substream，以及为什么 _direct_send_ 可以复用同一个 substream。

### Move Languages

_Move language_ 是不得不提的。即使 libra 这个项目无法继续存在下去，我相信 Move 仍然值得投入精力去发展，直到实现自身的自举。Libra 中，几乎所有用户看得见功能都通过 Move 实现，包括宣传的 Libra Account 和 Libra Coin。 

Move 目前包含了：

- 将 move script 编译成  move bytecode 的 compiler。

  script 中凡是涉及到对 resource 的操作，compiler 都会严格检查其 move 语义。比如： LibraCoin ，在制造后，只能从一个 account的 resource 中 move 到另外一个 account 的 resource 中，不会凭空消失，也不会凭空出现或者翻倍。这是 compile 最核心的功能。有 language 功底和形式化验证能力的童靴可深入了解这部分。如果 blockchain 能够发展起来， 对 contract 做形式化验证的场景会越来越多，甚至是决定了一个平台能否做大做强的重要因素，一个反面例子就是 ETH solidity contract。

- 执行 bytecode 的 runtime。 runtime 其实没啥好讲的，大部分都是老一套，建一个执行栈（没有堆），然后把指令一个一个加载进来，解析并执行，中间会涉及到栈顶元素的 pop 和 push。

- bytecode 指令集中提供了访问 libra storage 的数据接口，主要是对 account 下 resouce 的读取和写入：`borrow_global`, `move_to` 。 这是 move 作为 blockchain contact langauge 的重要特征。



#### 自举与软分叉

前面我提到了自举，其实不能叫自举。compiler 实现可以沿用现有的 rust 实现，甚至是用其他语言实现 compiler 也都没有问题。唯一重要的是，能够将 move bytecode 的 runtime 用 move language 实现。

为什么说重要：runtime 实现自举后，可以直接把 runtime 的 bytecode 放到链上去，这对不断 involve 的 move language 有很大的好处。 新的 runtime 可以直接通过共识更新到其他的节点上，保证节点无缝升级。我相信这种方式会成为主流。

#### 链下与链上

Move script 可以直接在链下执行，拿到对链上数据的_更新集合_。这个性质给 libra 周边生态带来了无限可能性，比如，状态通道可以基于它来搞。很多在链上执行非常消耗 gas 的合约，可以在线下执行完，合约涉及的多方在线下达成共识后，跳过了 gas 消耗，直接把最后的更新集合上链。相信还有很多其他有意思的东西可以基于 move script 来做。



## Rust 经验

在 Libra 之前，我只阅读过一个大型的 Rust 项目代码，PingCAP 的 TiKV，写 Rust 的机会更是少之又少。进入区块链行业，也是有想专职写 Rust 的目的。（虽然说语言不重要，但我就是 Rust 拥趸）。Libra 代码写的很大气，可以学而时习之。

- future，channel 实现 actor model。

- Rust trait，type parameter，associate types。逐渐认识到 rust 类型系统的强大之处，当然也还有些 features 待完善。

  比如：generic associate type，return `impl Trait` in trait methods。

- 学会和编译器做朋友，可以从编译器的提示中熟悉 Rust 的秉性。

- 最后，写 Rust 让我意识到，可能真是该换顶配 Mac 了。



最后一点经验是关于 RocksDB的，姑且也放在这里吧。RocksDB 我用了它的 prefix iterator 的特性，抽象出了 namespace 的功能，用来隔离不用namespace 的数据。

## 开发者能做什么

- 分布式的，可验证的 move contract 存储平台。当 Libra 发展到一定程度后，必然需要一个去中心化，可信的 move contract 存储。这种平台需要具有针对 contract 的评估机制，从而建立一个用户可信的智能合约发布和存储平台。这有点类似于各种语言的包管理平台，比如 Java 中的 Maven 仓库，Rust 的 crates.io，npm registry 等等，只不过它们都是中心化的，并且缺乏代码安全性审核和评估（Rust 社区正在搞一个类似的东西）。相比这些语言，move script 的评估会简单很多，毕竟 contract 的功能不会太过复杂。
- 针对 move contract 的形式化验证，其他合约语言也需要这样的形式化手段。
- 各种小应用，小游戏。比如可以在 libra 中实现五子棋游戏，扑克游戏。



[tokio-scheduler]: https://tokio.rs/blog/2019-10-scheduler/	"tokio scheduler got 10x improvements"






