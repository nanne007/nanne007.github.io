### build ###

install deps:

```
brew install cmake protobuf go
```

## Chain ##



## Move Language ##

concept:
- address: 256 bit bytes.
- resource: instance of resource type.
- module: contain resouce type defs and procedure defs.

### Bytecode verifier ###

- control-flow graph construction.
- stack balance checking.
- type checking.
- Kind checking: based on rules about resource types.
  - resource cannot be duplicated.
  - resource cannot be destroyed.
  - resource must be used.
- Reference checking. (will be another paper on this)
- Linking: check struct types and procedures.

### Move VM ###

基于全局状态，执行一个 tx，生成 tx 副作用（代表了对全局状态的修改，也是以 account_address -> account 的 map 形式）。

目前 tx 是顺序执行的。Move 支持并行执行，后续有改进空间。

每个 tx 有自己的 readset 和 writeset（副作用），当不同 tx 的 readset/writeset 没有交集，就可以并行执行。


### What's next ###


- 核心功能实现：accounts, coin, reserve management, validator node 的加入和退出，tx fee，冷钱包。
- 新语言特性。
- 用来更新 moudle 的可信机制。
- 形式验证。
- 三方 module 支持。
