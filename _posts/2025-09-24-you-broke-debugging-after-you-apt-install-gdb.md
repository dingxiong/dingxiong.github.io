---
layout: post
title: You break debugging after you install gdb
date: 2025-09-24 17:58 -0700
categories: [os, linux]
tags: [linux, linker, gdb]
---

A funny thing happened yesterday. I tried to use `memray` to debug memory leak
of a python service in production, but `memray attach <pid>` failed to work.
The reason is that I did `apt-get install gdb` before installing `memray`. Wait
a minute. `gdb` is a prerequisite for `memray attach`. How does gdb breaks
`memray`.

A little background about how `memray attach` works. Almost all these tools
rely on a debugger `gdb` or `lldb` to attach to the running process. Some
ambitious authors will write from scratch by only using `ptrace`, but it is
rare. `memray` is the former case. The source code is
[here](https://github.com/bloomberg/memray/blob/d8b6d005c4039e8850ac7906b5d606cad2d27747/src/memray/commands/_attach.gdb#L38).
The whole magic is just a gdb script. This script sets a bunch of breakpoints
at function `malloc`, `free`, `PyMem_Malloc` etc and registers a callback
function `memray_spawn_client`. This makes total sense for a memory profiler.
However, the tricky part is the line doing `dlopen`. For my case, the two
parameters were

```
$libpath = /usr/local/lib/python3.12/site-packages/memray/_memray.cpython-312-x86_64-linux-gnu.so
$rtld_now = 2
```

`2` means loading `_memray.cpython-312-x86_64-linux-gnu.so` immediately.

If `dlopen` succeeds, it should return a valid handler pointer, but what I got
was always `0x0`. I tried to add `dlerror()` after that line to see what error
was, but immediately got segmentation fault!

Is the problem with `_memray.cpython-312-x86_64-linux-gnu.so` itself? Chatgpt
told me to verify it by running below script.

```
gdb -p $PID -batch \
  -ex 'set unwindonsignal off' \
  -ex 'set $r = (void*) dlopen("libm.so.6", 2)' \
  -ex 'printf "dlopen -> %p\n", $r' \
  -ex quit
```

Fine. It printed `dlopne -> null`. What? It did not work for any shared
library. Maybe the problem was at OS level? Chatgpt quickly generated a simple
C program to call `dlopne("libm.so.6", 2)`. It ran without problem, so the
problem was with this specific `$PID` process. I had a few other hypothesis,
but none of them turned out to be the root cause. Then I casually read the
memory maps of this process. It was a long list! I lost my patience and threw
all of them to Chatgpt, and Chatgpt highlighted a few suspicious mappings.

```
0x7f75d0bed000     0x7f75d0bee000     0x1000        0x0  r--p   /usr/lib/x86_64-linux-gnu/libdl.so.2 (deleted)
0x7f75d0bee000     0x7f75d0bef000     0x1000     0x1000  r-xp   /usr/lib/x86_64-linux-gnu/libdl.so.2 (deleted)
0x7f75d0bef000     0x7f75d0bf0000     0x1000     0x2000  r--p   /usr/lib/x86_64-linux-gnu/libdl.so.2 (deleted)
0x7f75d0bf0000     0x7f75d0bf1000     0x1000     0x2000  r--p   /usr/lib/x86_64-linux-gnu/libdl.so.2 (deleted)
0x7f75d0bf1000     0x7f75d0bf2000     0x1000     0x3000  rw-p   /usr/lib/x86_64-linux-gnu/libdl.so.2 (deleted)
```

Not just `libdl` but also `libm.so` and a few other shared libs. Nice! This
must be related because I did not see `(deleted)` on my local host.

`(deleted)` means the file was deleted, but its content was still used by the
process and it exits in memory. Think about `mmap` and then deleting the
physical file. I checked the file path, it did exist in disk. I expressed my
confusion to Chatgpt, and it told me that the shared lib might be upgraded with
the file path kept unchanged. Chatgpt said I might have upgraded `libc`. I was
very suspicious about this answer because it was not the first time Chatgpt
lied to me.

I took a break for two hours because my daughter would not sleep if I did not
sleep at the same time. I closed my eyes lying on the bed and thought about
what I had done after logging into this machine. I did `apt-get install procps`
because I wanted to use `ps`, then did `apt-get installl gdb` and
`uv pip install memray` and `apt-get install neovim` because I wanted to write
some C code to test my hypothesis. Nothing I could think of to be related to
`libc` or `ld`.

Wait a minute! `apt-get install gdb`! I found my daughter was already asleep,
and I went to my laptop, logged into another similar production box, and type
`apt-get install --simulate gdb`. It actually automatically upgrades `glibc`!
Holy shit. I quickly compiled `gdb` from source, and then ran
`memray attach <pid>`. And it worked!

This is the full story. I know this isn't significant at all in many people's
view. We are never short of gotchas in the Linux world. But I kind of feel this
is a good lesson for me. I started to appreciate the difficulty of problems
that Arch Linux faces. It is not easy to do rolling upgrade for long-running
servers. Meanwhile, there is still some puzzle I do not fully resolve.
`(delete)` means file was deleted/updated on the disk, but still memory has it,
so in principle, it should continue to work, but why `gdb` must need the
physical existence. Moreover, I could not find any manual/documentation
discussing this gotcha. `gdb` relying on physical existence is just my
guess/observation!
