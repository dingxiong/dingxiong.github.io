---
layout: post
title: Rust -- Getting Started
date: 2024-10-14 00:30 -0700
categories: [programming-language, rust]
tags: [rust]
---

## Macros

What is a procedural macro? See
<https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros>

One example is `#[tokio::main]`. See implementation
[here](https://github.com/tokio-rs/tokio/blob/512e9decfb683d22f4a145459142542caa0894c9/tokio-macros/src/lib.rs#L254).
For a simple program like below,

```rust
#[tokio::main]
async fn main() {
    let result = some_async_function().await;
    println!("Result: {:?}", result);
}

async fn some_async_function() -> i32 {
    42
}
```

we run `cargo expand` and the result is below.

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
fn main() {
    let body = async {
        let result = some_async_function().await;
        {
            ::std::io::_print(format_args!("Result: {0:?}\n", result));
        };
    };
    #[allow(clippy::expect_used, clippy::diverging_sub_expression)]
    {
        return tokio::runtime::Builder::new_multi_thread()
            .enable_all()
            .build()
            .expect("Failed building the Runtime")
            .block_on(body);
    }
}
async fn some_async_function() -> i32 {
    42
}
```
