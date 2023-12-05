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
