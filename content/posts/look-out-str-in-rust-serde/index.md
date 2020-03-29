---
title: "serde 反序列化 str 引发的血案"
date: 2020-03-29
---

serde 是 Rust 中一个被广泛使用的序列化框架。本文给出我最近遇到的一个反序列化的坑。

区块链中，public key 展现出来的是一串字符串（无论用什么编码，提供给 human read 的总归是字符串）。
但是程序需要将这些字符串转换成语言内部表示的数据结构。

libra 也是这样，它使用 serde 为 PublicKey 定制了一个反序列化方式。

```rust
        impl<'de> ::serde::Deserialize<'de> for #name {
            fn deserialize<D>(deserializer: D) -> std::result::Result<Self, D::Error>
            where
                D: ::serde::Deserializer<'de>,
            {
                if deserializer.is_human_readable() {
                    let encoded_key = <&str>::deserialize(deserializer)?;
                    ValidKeyStringExt::from_encoded_string(encoded_key)
                        .map_err(<D::Error as ::serde::de::Error>::custom)
                } else {
                   // ignored code....
                }
            }
        }
```

这段代码试图从 `deserializer` 中拿到一段 `&'de str`，然后再从 `&str` 中生成 PublicKey。

遇到的坑就是 **从 `deserializer` 中拿 `&'de str`** 时，会报错 `expect borrowed string`。

重点在于，只有当你用了某些类型的 `deserializer`，才会报错。

下面这段代码是可以正常执行的。直接 *plaintext => publickey*
```rust
let origin_text = "731fe437a8d3fbb25fa389307ac615e3a503e49be40e1b8cf9e5136fb44b9e5f";
let public_key: PublicKey = Deserialize::deserialize(origin_text).unwrap();
```

但是 *plaintext => jsonvalue => publickey* ，从 jsonvalue 转换成 publickey 出现了问题。

```rust
let origin_text = "731fe437a8d3fbb25fa389307ac615e3a503e49be40e1b8cf9e5136fb44b9e5f";
let json_value = serde_json::from_str(origin_text).unwrap();
let public_key: PublicKey = Deserialize::deserialize(json_value).unwrap();
```

问题就出在，无法从 `Json::String(v)` 反序列化出一个 `&str`。

因为 json_value 在 `Deserialize::deserialize(deserializer: D)` 调用会将 `deserializer` move 进去，

调用完成之后，`deserializer` 不存在了，`Json::String(v)` 中的 `v` 也就没有了，那返回的 `&str` 该引用哪一段内存块呢？

反序列化时， serde_json 对 `&str` 和 `String` 的处理方式一样，都是调用 `Visitor` 的 `visit_string` 方法，

``` rust

    fn deserialize_str<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>,
    {
        self.deserialize_string(visitor)
    }

    fn deserialize_string<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>,
    {
        match self {
            Value::String(v) => visitor.visit_string(v),
            _ => Err(self.invalid_type(&visitor)),
        }
    }


```

但是！两者的 visitor 不同。`&str` 对应的 `Visitor` 是 `StrVisitor`。

```rust
struct StrVisitor;

impl<'a> Visitor<'a> for StrVisitor {
    type Value = &'a str;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("a borrowed string")
    }

    fn visit_borrowed_str<E>(self, v: &'a str) -> Result<Self::Value, E>
    where
        E: Error,
    {
        Ok(v) // so easy
    }

    fn visit_borrowed_bytes<E>(self, v: &'a [u8]) -> Result<Self::Value, E>
    where
        E: Error,
    {
        str::from_utf8(v).map_err(|_| Error::invalid_value(Unexpected::Bytes(v), &self))
    }
}

impl<'de: 'a, 'a> Deserialize<'de> for &'a str {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_str(StrVisitor)
    }
}
```


`StrVisitor` 没有实现 `visit_string` 方法，使用 `Visitor` 的默认实现。而默认实现，就是直接报错。

``` rust
    fn visit_str<E>(self, v: &str) -> Result<Self::Value, E>
    where
        E: Error,
    {
        Err(Error::invalid_type(Unexpected::Str(v), &self))
    }
    #[inline]
    #[cfg(any(feature = "std", feature = "alloc"))]
    fn visit_string<E>(self, v: String) -> Result<Self::Value, E>
    where
        E: Error,
    {
        self.visit_str(&v)
    }
```

## 解决办法

这里的坑在于，实现 `Deserialize` 时，无法预知，`deserializer` 是否支持 `&str`(&[u8] 同理) 的反序列化。
而我们想要的是，如果支持，就返回 *borrowed str*，如果不支持，返回*owned str*。
而这，可以用 `Cow<str>` 实现。

serde 本身对 `Cow<T>` 的 `Deserialize` 实现，是直接返回 `Cow::Owned`。

但 serde 在实现 derive 时，考虑了 `str` 和 `[u8]` 两种特殊情形，专门提供了 `borrow_cow_str` 和 `borrow_cow_bytes` 两种方法，
可以返回 `Cow::Borrowed` 数据。

```
#[cfg(any(feature = "std", feature = "alloc"))]
pub fn borrow_cow_str<'de: 'a, 'a, D, R>(deserializer: D) -> Result<R, D::Error>
where
    D: Deserializer<'de>,
    R: From<Cow<'a, str>>,
{
    struct CowStrVisitor;

    impl<'a> Visitor<'a> for CowStrVisitor {
        type Value = Cow<'a, str>;

        fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
            formatter.write_str("a string")
        }

        fn visit_str<E>(self, v: &str) -> Result<Self::Value, E>
        where
            E: Error,
        {
            Ok(Cow::Owned(v.to_owned()))
        }

        fn visit_borrowed_str<E>(self, v: &'a str) -> Result<Self::Value, E>
        where
            E: Error,
        {
            Ok(Cow::Borrowed(v))
        }

        fn visit_string<E>(self, v: String) -> Result<Self::Value, E>
        where
            E: Error,
        {
            Ok(Cow::Owned(v))
        }

        // ignored other code ...
    }

    deserializer.deserialize_str(CowStrVisitor).map(From::from)
}

```

因此，实现 `&str`  序列化时，最好使用上面这个方法。
libra 实现可以改成：

```rust
let encoded_key: Cow<str> = serde::private::borrow_cow_str(deserializer)?;
```
