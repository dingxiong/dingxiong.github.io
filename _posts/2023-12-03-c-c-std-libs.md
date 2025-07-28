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

## glibc

Glibc is the standard C library for the GNU operating system and other systems
that use the Linux kernel. For a long time, I am confused by what exactly glibc
provides because I saw lots of code that include headers such as
`<sys/epoll.h>` and `<unistd.h>`. I definitely did not see these headers when I
was learning C as a undergraduate. Cited from glibc's
[home page](https://www.gnu.org/software/libc/)

> The GNU C Library - The project provides the core libraries for the GNU
> system and GNU/Linux systems, as well as many other systems that use Linux as
> the kernel. These libraries provide critical APIs including ISO C11,
> POSIX.1-2008, BSD, OS-specific APIs and more. These APIs include such
> foundational facilities as open, read, write, malloc, printf, getaddrinfo,
> dlopen, pthread_create, crypt, login, exit and more.

So basically, it not only supports ISO C, but also POSIX.1 standard. And there
are quite a few Linux specific system calls. The C standard header files are
[here](https://en.cppreference.com/w/c/header). POSIX.1 header files are
[here](https://en.wikipedia.org/wiki/C_POSIX_library). The headers provided by
glibc is under the
[include directory](https://github.com/bminor/glibc/tree/master/include). There
are much more headers inside `include/sys` than specified by POSIX.1. Most of
them are wrappers to Linux system calls.

```
Header                             POSIX header
-----------------------------------------------------------
include/sys/ipc.h                    Y
include/sys/mman.h                   Y
include/sys/msg.h                    Y
include/sys/resource.h               Y
include/sys/select.h                 Y
include/sys/sem.h                    Y
include/sys/shm.h                    Y
include/sys/socket.h                 Y
include/sys/statvfs.h                Y
include/sys/stat.h                   Y
include/sys/times.h                  Y
include/sys/time.h                   Y
include/sys/types.h                  Y
include/sys/uio.h                    Y
include/sys/un.h                     Y
include/sys/utsname.h                Y
include/sys/wait.h                   Y
-----
include/sys/unistd.h                 N
include/sys/ttychars.h               N
include/sys/timeb.h                  N
include/sys/termios.h                N
include/sys/sysmacros.h              N
include/sys/syslog.h                 N
include/sys/sysinfo.h                N
include/sys/statfs.h                 N
include/sys/single_threaded.h        N
include/sys/signal.h                 N
include/sys/sendfile.h               N
include/sys/resource.h               N
include/sys/random.h                 N
include/sys/queue.h                  N
include/sys/profil.h                 N
include/sys/prctl.h                  N
include/sys/poll.h                   N
include/sys/param.h                  N
include/sys/ioctl.h                  N
include/sys/gmon_out.h               N
include/sys/gmon.h                   N
include/sys/file.h                   N
include/sys/fcntl.h                  N
include/sys/errno.h                  N
include/sys/epoll.h                  N
include/sys/dir.h                    N
include/sys/cdefs.h                  N
include/sys/bitypes.h                N
include/sys/auxv.h                   N
include/sys/xattr.h                  N
include/sys/vlimit.h                 N
include/sys/vfs.h                    N
```

### build

Building glibc in Linux is straightforward.

```bash
git clone git://sourceware.org/git/glibc.git --depth=1
cd glibc
mkdir -p build
cd build
../configure --prefix /home/admin/code/glibc/build
bear -- make -j
cd ..
ln -s build/compile_commands.json compile_commands.json
```

The `bear make` step takes a long time. Be patient!

I failed to cross compile it in Macbook M1.

### backtrace in glibc

There are multiple ways to get stacktraces in C/C++. This is great
[blog](https://eli.thegreenplace.net/2015/programmatic-access-to-the-call-stack-in-c)
on this topic. It claims there are three ways to obtain a stacktrace. First is
the gcc's builtin function `__builtin_return_address`. I tried to read relevant
code in gcc, but quickly got lost as the implementation of this builtin
function finally resorts to register manipulation.

Let's talk about the third option `libunwind` then. It claims to be portable
and efficient to unwind stack traces. Let's focus on the backtrace
functionality. The core code is in file
[Backtrace.c](https://github.com/libunwind/libunwind/blob/05afdabf38d3fa461b7a9de80c64a6513a564d81/src/unwind/Backtrace.c#L28)
which defines function `_Unwind_Backtrace`.

The second's option is `glibc`'s
[backtrace](https://github.com/bminor/glibc/blob/cf11e74b0d81d389bcad5cdbba020ba475f0ac4b/debug/backtrace.c#L64)
function. Note, the backtrace symbol is exposed by
`weak_alias (__backtrace, backtrace)`. This function is simple. It calls
[\_\_libc_unwind_link_get()](https://github.com/bminor/glibc/blob/cf11e74b0d81d389bcad5cdbba020ba475f0ac4b/misc/unwind-link.c#L41)
to get the real loaded shared library which contains a symbol
`_Unwind_Backtrace`. So basically `backtrace` function delegates all work to
this shared library. The shared library is called `LIBGCC_S_SO`. I searched the
whole repo but cannot find the definition of this macro. After compiling glibc,
I find a file called `lib-names-64.h` with below content

```
/* This file is automatically generated.  */
#ifndef __GNU_LIB_NAMES_H
# error "Never use <gnu/lib-names-64.h> directly; include <gnu/lib-names.h> instead."
#endif

#define LD_LINUX_X86_64_SO              "ld-linux-x86-64.so.2"
#define LD_SO                           "ld-linux-x86-64.so.2"
#define LIBANL_SO                       "libanl.so.1"
#define LIBBROKENLOCALE_SO              "libBrokenLocale.so.1"
#define LIBC_MALLOC_DEBUG_SO            "libc_malloc_debug.so.0"
#define LIBC_SO                         "libc.so.6"
#define LIBDL_SO                        "libdl.so.2"
#define LIBGCC_S_SO                     "libgcc_s.so.1"
#define LIBMVEC_SO                      "libmvec.so.1"
#define LIBM_SO                         "libm.so.6"
#define LIBNSL_SO                       "libnsl.so.1"
#define LIBNSS_COMPAT_SO                "libnss_compat.so.2"
#define LIBNSS_DB_SO                    "libnss_db.so.2"
#define LIBNSS_DNS_SO                   "libnss_dns.so.2"
#define LIBNSS_FILES_SO                 "libnss_files.so.2"
#define LIBNSS_HESIOD_SO                "libnss_hesiod.so.2"
#define LIBNSS_LDAP_SO                  "libnss_ldap.so.2"
#define LIBPTHREAD_SO                   "libpthread.so.0"
#define LIBRESOLV_SO                    "libresolv.so.2"
#define LIBRT_SO                        "librt.so.1"
#define LIBTHREAD_DB_SO                 "libthread_db.so.1"
#define LIBUTIL_SO                      "libutil.so.1"
```

So this shared library is called `libgcc_s.so.1`. It is the
[GCC low-level runtime function](https://gcc.gnu.org/onlinedocs/gccint/Libgcc.html).
OK, so let's see what is inside this .so file.

```
> gcc -print-file-name=libgcc_s.so.1
/usr/lib/gcc/x86_64-linux-gnu/10/../../../x86_64-linux-gnu/libgcc_s.so.1

> readlink -f /usr/lib/gcc/x86_64-linux-gnu/10/../../../x86_64-linux-gnu/libgcc_s.so.1
/usr/lib/x86_64-linux-gnu/libgcc_s.so.1

> nm -D /usr/lib/gcc/x86_64-linux-gnu/10/../../../x86_64-linux-gnu/libgcc_s.so.1
...
0000000000011250 T _Unwind_Backtrace@@GCC_3.3
0000000000011230 T _Unwind_DeleteException@@GCC_3.0
000000000000e8b0 T _Unwind_FindEnclosingFunction@@GCC_3.3
0000000000013180 T _Unwind_Find_FDE@@GCC_3.0
0000000000010e20 T _Unwind_ForcedUnwind@@GCC_3.0
000000000000e7e0 T _Unwind_GetCFA@@GCC_3.3
000000000000e8d0 T _Unwind_GetDataRelBase@@GCC_3.0
000000000000e7a0 T _Unwind_GetGR@@GCC_3.0
000000000000e850 T _Unwind_GetIP@@GCC_3.0
000000000000e860 T _Unwind_GetIPInfo@@GCC_4.2.0
000000000000e890 T _Unwind_GetLanguageSpecificData@@GCC_3.0
000000000000e8a0 T _Unwind_GetRegionStart@@GCC_3.0
000000000000e8e0 T _Unwind_GetTextRelBase@@GCC_3.0
0000000000010a90 T _Unwind_RaiseException@@GCC_3.0
0000000000010fc0 T _Unwind_Resume@@GCC_3.0
0000000000011150 T _Unwind_Resume_or_Rethrow@@GCC_3.3
000000000000e7f0 T _Unwind_SetGR@@GCC_3.0
000000000000e880 T _Unwind_SetIP@@GCC_3.0
```

First `gcc -print-file-name` command is so useful! Second, the file path is so
wired. It is inside `/urs/lib/x86_64-linux-gnu`, but not `/usr/lib/gcc`. I
suspect it is shipped with Linux instead of installed together with gcc. So I
did a quick experiment. I launched a docker container
`docker run -i -t ubuntu /bin/bash` and inside this fresh ubuntu image

```
root@6adc2dc89d0b:/# ll /usr/lib/aarch64-linux-gnu/libgcc_s.so.1
-rw-r--r-- 1 root root 84296 May 13  2023 /usr/lib/aarch64-linux-gnu/libgcc_s.so.1
```

This .so file exits without installing gcc!

Third, this shared library has quite a few function starting with `_Unwind_`.
And you notice that all these functions show up in `libunwind` library as well.
So basically, this is glibc's own implementation of `libunwind`? Actually, it
depends. glibc has its own implementation such as
[this one](https://github.com/gcc-mirror/gcc/blob/d1a21a6f9474e519926d20a7c6d664be03aff3ee/libgcc/unwind.inc#L291).
But it is not always used. It depends on the flag
[USE_LIBUNWIND_EXCEPTIONS](https://github.com/gcc-mirror/gcc/blob/d1a21a6f9474e519926d20a7c6d664be03aff3ee/gcc/gcc.cc#L1926).
If this flag is defined, GCC will link with `-lunwind` and all its internal
`_Unwind_` implementations delegate to corresponding ones in `libunwind`.
glics's own doc has clear statement about this well. See
[doc](https://github.com/bminor/glibc/blob/cf11e74b0d81d389bcad5cdbba020ba475f0ac4b/manual/debug.texi#L42).
