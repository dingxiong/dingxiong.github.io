---
layout: post
title: Rust -- Getting Started
date: 2024-10-14 00:30 -0700
categories: [programming-language, rust]
tags: [rust]
---

## Setup for newbie

1. Install `rustup`.
2. Install `rust-analyzer`: `rustup component add rust-analyzer` and config
   nvim lspconfig and treesitter.
3. Read <https://doc.rust-lang.org/stable/book/>
4. Also checkout <https://rustc-dev-guide.rust-lang.org/getting-started.html>

## Build from source

See the
[official doc](https://rustc-dev-guide.rust-lang.org/building/quickstart.html).

```
git clone https://github.com/rust-lang/rust.git --depth=1
cd rust
./x setup
./x build
```

It takes a long time because it compile LLVM as well. It also uses a lot of
disk. After build finishes, verify that the new toolchain is available.

```
$ rustup toolchain list -v
stable-aarch64-apple-darwin (default)   /Users/xiongding/.rustup/toolchains/stable-aarch64-apple-darwin
1.72.0-aarch64-apple-darwin     /Users/xiongding/.rustup/toolchains/1.72.0-aarch64-apple-darwin
stage1  /Users/xiongding/code/rust/build/aarch64-apple-darwin/stage1
```

Rebuild is slow as hell. So most time we just need incremental build

```
./x build library --keep-stage 1
```

Now, we can build rust code with this new compiler.

```
rustc +stage1 test.rs -o a.out
```

### Enable debug logs

There are a lot of debug/trace logs inside rustc that can be useful for
learning the internals of rustc. For example,
[this one](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_resolve/src/imports.rs#L767).
How to enable it?

First, the debug/trace log macros are defined in the
[tokio-rs/tracing](https://github.com/tokio-rs/tracing) library, a 3rd party
library. When building rustc, we need to turn on the debug feature of this
library in `cargo.toml`.

```
tracing = { version = "0.1", features = ["max_level_debug", "release_max_level_debug"] }
```

Second, when running rustc, we should set the `RUSTC_LOG` environment variable.
See [official doc](https://rustc-dev-guide.rust-lang.org/tracing.html).

```
RUSTC_LOG=rustc_resolve::import=debug rustc +stage1 test.rs -o a.out
```

## What is a module?

See
[definition](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_resolve/src/lib.rs#L491).
Any code block is a module.

## What is a namespace?

Types live in the type namespace. Variables live in the variable namespace. See
[definition](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_hir/src/def.rs#L541).

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

## Lifetime Parameter

## Name Resolution

`use` statement brings definitions in other modules into the current scope.
Inside `rustc`, it is called `import`. See its parser
[here](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_parse/src/parser/item.rs#L197).
`rustc_resolve` module is responsible for name resolution during compilation.
Its primary job is to resolve paths, names, and items used in a Rust program to
their corresponding definitions within the scope of the code.

Use/import is an important aspect of name resolution. At what step is name
resolution executed? For a small program below, let's see when `use math:xxxxx`
is resolved.

```rust
mod math {
    pub fn xxxxx(a: i32, b: i32) -> i32 {
        a + b
    }
}
fn main() {
    use math::xxxxx;
    let sum = xxxxx(5, 3);
    println!("The sum of 5 and 3 is: {}", sum);
}
```

I added a line to print out backtrace
[here](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_resolve/src/imports.rs#L767).
Below is the output.

```
   0: std::backtrace::Backtrace::create
   1: <rustc_resolve::Resolver as rustc_expand::base::ResolverExpand>::resolve_imports
   2: <rustc_expand::expand::MacroExpander>::fully_expand_fragment
   3: <rustc_expand::expand::MacroExpander>::expand_crate
   4: <rustc_session::session::Session>::time::<rustc_ast::ast::Crate, rustc_interface::passes::configure_and_expand::{closure#1}>
   5: rustc_interface::passes::resolver_for_lowering_raw
   6: rustc_query_impl::plumbing::__rust_begin_short_backtrace::<rustc_query_impl::query_impl::resolver_for_lowering_raw::dynamic_query::{closure#2}::{closure#0}, rustc_middle::query::erase::Erased<[u8; 16]>>
   7: <rustc_query_impl::query_impl::resolver_for_lowering_raw::dynamic_query::{closure#2} as core::ops::function::FnOnce<(rustc_middle::ty::context::TyCtxt, ())>>::call_once
   8: rustc_query_system::query::plumbing::try_execute_query::<rustc_query_impl::DynamicConfig<rustc_query_system::query::caches::SingleCache<rustc_middle::query::erase::Erased<[u8; 16]>>, false, false, false>, rustc_query_impl::plumbing::QueryCtxt, false>
   9: rustc_query_impl::query_impl::resolver_for_lowering_raw::get_query_non_incr::__rust_end_short_backtrace
  10: <rustc_middle::ty::context::TyCtxt>::resolver_for_lowering
  11: <rustc_interface::queries::QueryResult<&rustc_middle::ty::context::GlobalCtxt>>::enter::<&rustc_data_structures::steal::Steal<(rustc_middle::ty::ResolverAstLowering, alloc::sync::Arc<rustc_ast::ast::Crate>)>, rustc_driver_impl::run_compiler::{closure#0}::{closure#1}::{closure#3}>
  12: <rustc_interface::interface::Compiler>::enter::<rustc_driver_impl::run_compiler::{closure#0}::{closure#1}, core::result::Result<core::option::Option<rustc_interface::queries::Linker>, rustc_span::ErrorGuaranteed>>
  13: rustc_span::create_session_globals_then::<core::result::Result<(), rustc_span::ErrorGuaranteed>, rustc_interface::util::run_in_thread_with_globals<rustc_interface::util::run_in_thread_pool_with_globals<rustc_interface::interface::run_compiler<core::result::Result<(), rustc_span::ErrorGuaranteed>, rustc_driver_impl::run_compiler::{closure#0}>::{closure#1}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#0}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#0}::{closure#0}::{closure#0}>
  14: std::sys::backtrace::__rust_begin_short_backtrace::<rustc_interface::util::run_in_thread_with_globals<rustc_interface::util::run_in_thread_pool_with_globals<rustc_interface::interface::run_compiler<core::result::Result<(), rustc_span::ErrorGuaranteed>, rustc_driver_impl::run_compiler::{closure#0}>::{closure#1}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#0}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#0}::{closure#0}, core::result::Result<(), rustc_span::ErrorGuaranteed>>
  15: <<std::thread::Builder>::spawn_unchecked_<rustc_interface::util::run_in_thread_with_globals<rustc_interface::util::run_in_thread_pool_with_globals<rustc_interface::interface::run_compiler<core::result::Result<(), rustc_span::ErrorGuaranteed>, rustc_driver_impl::run_compiler::{closure#0}>::{closure#1}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#0}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#0}::{closure#0}, core::result::Result<(), rustc_span::ErrorGuaranteed>>::{closure#1} as core::ops::function::FnOnce<()>>::call_once::{shim:vtable#0}
  16: std::sys::pal::unix::thread::Thread::new::thread_start
  17: __pthread_deallocate

 DEBUG rustc_resolve::imports (resolving import for module) resolving import `math::...` in `<opaque>`
```

Hm, imports are resolved inside macro expansion `fully_expand_fragment`. The
[comment](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_expand/src/expand.rs#L435)
says

> Optimization: if we resolve all imports now, we'll be able to immediately
> resolve most of imported macros.

So basically, resolving names can help macros. It makes sense that the macro
may be defined in the imported module. The
[rustc dev guide](https://rustc-dev-guide.rust-lang.org/macro-expansion.html)
clearly says the first step in the macro expansion loop is resolving imports.

Second question is what happens exactly after the use statement is executed.

### Rib

https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_resolve/src/late.rs#L4481
