---
title: C/C++ std libs
date: 2023-12-03 23:16 -0800
categories: [programming-language, cpp]
tags: [compiler, cpp, gcc, llvm, glibc, libstdc++, libc++]
---

Recently, I was revisiting the C++ `std::time` function and suddenly get
triggered by the `std::time_t` type. Quoted from
[cppreference](https://en.cppreference.com/w/cpp/chrono/c/time_t),

> Although not defined, this is almost always an integral value holding the
> number of seconds (not counting leap seconds) since 00:00, Jan 1 1970 UTC,
> corresponding to POSIX time

I have visited this page many times in my career life, but this is the first
time I go deeper into it. This type is only declared by C++ standard, but not
defined by C++ standard. Then where is it defined?

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

## Libc and glibc

Before we answer the question where `std::time_t` is defined, let's take a look
at the function `std::time_t std::time(std::time_t *)` in `ctime` header.

First In `libc++`, the declaration is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/ctime#L75).
If I click `go to definition`, my IDE jumps to file
`/Library/Developer/CommandLineTools/SDKs/MacOSX14.0.sdk/usr/include/time.h`,
which is
[here](https://github.com/apple-oss-distributions/Libc/blob/main/include/time.h#L163).
This repo `Libc` is the standard C library provided by MacOS.

In `libstdc++`, the `std::time` function is declared
[here](https://github.com/gcc-mirror/gcc/blob/88029286c35d3bf65568fea1324d595a15441772/libstdc++-v3/include/c/ctime#L31-L32)

```cpp
#pragma GCC system_header
#include_next <time.h>
```

Actually, it is not defined at all. It is imported from somewhere.
`include_next` means please use the second `time.h` file in the search path.
Each C++ program will link with `glibc` as well. And `time.h` header is defined
[here](https://github.com/bminor/glibc/blob/master/time/time.h#L76).

To sum up, under `libc++` and `libstdc++` are `Libc` and `glibc`. They
implement the C standard.

## Operating systems

We still haven't answered the question where `std::time_t` is defined.

In my Macbook with Apple chip, the definition of `std::time_t` is in
`/Library/Developer/CommandLineTools/SDKs/MacOSX14.0.sdk/usr/include/sys/_types/_time_t.h`.

```
typedef __darwin_time_t         time_t;
```

Also, `__darwin_time_t` is defined in another file
`/Library/Developer/CommandLineTools/SDKs/MacOSX14.0.sdk/usr/include/arm/_types.h`,

```
typedef long                    __darwin_time_t;        /* time() */
```

Ah! `std::time_t` is just `long`. This type may be different in your laptop
because your laptop may have a different architecture. So we know `std::time_t`
is defined by the operating system.

So I keep showing the header files in my local laptop, but where are the source
code? The answer is "XNU", i.e., "X is Not Unix.", namely, the MacOS operating
system kernel. For example, the above definition of `__darwin_time_t` is
[here](https://github.com/apple-oss-distributions/xnu/blob/1031c584a5e37aff177559b9f69dbd3c8c3fd30a/bsd/arm/_types.h#L98).

How about Linux? Below is what I found in a Linux docker container.

```bash
$ cat /usr/include/x86_64-linux-gnu/sys/time.h  | grep time_t
#include <bits/types/time_t.h>

$ cat /usr/include/x86_64-linux-gnu/bits/types/time_t.h
typedef __time_t time_t;


$ cat /usr/include/x86_64-linux-gnu/bits/types.h | grep __time_t
__STD_TYPE __TIME_T_TYPE __time_t;      /* Seconds since the Epoch.  */

$ cat /usr/include/x86_64-linux-gnu/bits/typesizes.h | grep __TIME_T_TYPE
#define __TIME_T_TYPE           __SYSCALL_SLONG_TYPE

$ cat /usr/include/x86_64-linux-gnu/bits/typesizes.h | grep __SLONGWORD_TYPE
# define __SYSCALL_SLONG_TYPE   __SLONGWORD_TYPE

$ cat /usr/include/x86_64-linux-gnu/bits/types.h | grep __SLONGWORD_TYPE
#define __SLONGWORD_TYPE        long int
```

Ah! OK. `std::time_t` is defined as `long int` in Linux. You can find this file
in the Linux github repo as well.

We have gone so far in finding the definition of `std::time_t`!
