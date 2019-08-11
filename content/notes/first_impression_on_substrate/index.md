---
title: "Substrate 第一印象"
date: 2019-08-10T18:08:50+08:00
---

很早之前就听过 substrate 的大名，polkadot 也是基于 substrate 框架开发的，subsrate 想做一个通用的 blockchain 开发框架。
之前我也有过类似的想法。这么多的公链，讲道理 network，storage，consensus 等大部分的模块都是可以抽象出来，提供统一的接口供开发者做具体的实现。
这是题外话，说到 substrate，第一印象就是，WTF，怎么怎么多 macro？WTF，我的代码还要写在 macro 里面？WTF，这 tm 还是 Rust 吗？

看完下面这种代码。

``` rust
decl_module! {
	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
		/// The minimum period between blocks. Beware that this is different to the *expected* period
		/// that the block production apparatus provides. Your chosen consensus system will generally
		/// work with this to determine a sensible block time. e.g. For Aura, it will be double this
		/// period on default settings.
		const MinimumPeriod: T::Moment = T::MinimumPeriod::get();

		/// Set the current time.
		///
		/// This call should be invoked exactly once per block. It will panic at the finalization
		/// phase, if this call hasn't been invoked by that time.
		///
		/// The timestamp should be greater than the previous one by the amount specified by
		/// `MinimumPeriod`.
		///
		/// The dispatch origin for this call must be `Inherent`.
		#[weight = SimpleDispatchInfo::FixedOperational(10_000)]
		fn set(origin, #[compact] now: T::Moment) {
			ensure_none(origin)?;
			assert!(!<Self as Store>::DidUpdate::exists(), "Timestamp must be updated only once in the block");
			assert!(
				Self::now().is_zero() || now >= Self::now() + T::MinimumPeriod::get(),
				"Timestamp must increment by at least <MinimumPeriod> between sequential blocks"
			);
			<Self as Store>::Now::put(now.clone());
			<Self as Store>::DidUpdate::put(true);

			<T::OnTimestampSet as OnTimestampSet<_>>::on_timestamp_set(now);
		}

		fn on_finalize() {
			assert!(<Self as Store>::DidUpdate::take(), "Timestamp must be updated once in the block");
		}
	}
}
```

以及 `decl_module!` 的实现： https://github.com/paritytech/substrate/blob/dcbe1be4ebb88c6931c79ca93e36c4374bdde23f/srml/support/src/dispatch.rs#L215

已经欲哭无泪了。

容我再看两天。
