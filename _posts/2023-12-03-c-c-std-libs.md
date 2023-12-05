---
title: C/C++ std libs
date: 2023-12-03 23:16 -0800
categories: [programming-language, cpp]
tags: [compiler, c++, gcc, llvm. glibc]
---

Recently, I was revisiting the C++ `std::time` function and suddenly get
triggered by the `std::time_t` type. Quoted from
[cppreference](https://en.cppreference.com/w/cpp/chrono/c/time_t),

> Although not defined, this is almost always an integral value holding the
> number of seconds (not counting leap seconds) since 00:00, Jan 1 1970 UTC,
> corresponding to POSIX time

I visited this page many time in my career life, but this is the first time I
go deeper into it. This type is only declared by C++ standard, but not defined
by C++ standard. Then where is it defined?

## libc++ and libstdc++

We know there are three main C++ compilers: GCC, LLVM and Microsoft Visual C++
compiler. Each of them has their own implementation of the C++ standard
library. Gcc's implementation is called `libstdc++` and the source code is
[here](https://github.com/gcc-mirror/gcc/tree/master/libstdc%2B%2B-v3). LLVM's
implementation is called `libc++` and the source code is
[here](https://github.com/llvm/llvm-project/tree/main/libcxx).

For example, `std::vector` implementation is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/vector)
in `libc++`. The `libstdc++` version is more complicated. It is scattered in a
few places:
[here](https://github.com/gcc-mirror/gcc/blob/88029286c35d3bf65568fea1324d595a15441772/libstdc++-v3/include/std/vector#L66),
[here](https://github.com/gcc-mirror/gcc/blob/88029286c35d3bf65568fea1324d595a15441772/libstdc++-v3/include/bits/stl_vector.h#L1805-L1806)
and
[here](https://github.com/gcc-mirror/gcc/blob/88029286c35d3bf65568fea1324d595a15441772/libstdc++-v3/include/bits/vector.tcc#L1197).
As you can see that files under the `bits` subfolder make up the
implementation.
