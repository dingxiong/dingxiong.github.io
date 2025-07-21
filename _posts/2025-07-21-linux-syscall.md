---
layout: post
title: Linux -- Syscall
date: 2025-07-21 00:48 -0700
categories: [os, linux]
tags: [linux, memory]
---

## Kernel vs User Space

Kernel threads need memory as well. Where is it mapped to? In modern
architecture, kernel and user virtual memory space are collocated. However, the
same "kernel space" is mapped to all processes. For Linux x86_64 and Arm64, the
result is below.

```
| Platform | User Virtual Address Range              | Kernel Virtual Address Range            |
| -------- | --------------------------------------- | --------------------------------------- |
| x86_64   | 0x0000000000000000 – 0x00007fffffffffff | 0xffff800000000000 – 0xffffffffffffffff |
| arm64    | 0x0000000000000000 – 0x0000ffffffffffff | 0xffff000000000000 – 0xffffffffffffffff |
```

You may notice the math is obviously off. There is big hole between user space
and kernel space. There is a concept called `canonical addresses`. For x86_64,
the most significant bits (MSBs) of a canonical address must be a sign
extension of bit 47 (or bit 56 for 5-level paging). This means that the upper
bits (beyond the 48th bit or 57th bit in the case of 5-level paging) must
either be all 0s or all 1s, corresponding to the value of the 48th or 57th bit,
respectively. See some references
[here](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt).

By the way, `/prod/pid/maps` only shows the user space layout. Kernel space
layout is not accessible.

This setup boots performance. Being in the same virtual memory space avoids
switching between two sets of page tables, thus avoids TLB flushes.
Historically, for 32bit Linux, there is a patch to makes kernel space to be in
a separate space, but the performance penalty can be as high as 30%. See
[lwn article](https://lwn.net/Articles/39283/).

## Syscall implementation.

Historically, `syscall` is implemented using `INT` instruction in x86_64.
Nowadays, it uses `syscall` instruction instead. The latter is a lot of more
performant. In the arm world, it uses `svc` instruction.

```asm
# x86_64 example
# compile:
# as -o test.o test.s
# ld -o test test.o

.global _start

.text
_start:
    # write(1, message, 13)
    mov $1, %rax           # syscall number for write (SYS_write) is 1
    mov $1, %rdi           # file descriptor 1 is stdout
    lea message(%rip), %rsi # address of string to output (RIP-relative addressing)
    mov $13, %rdx          # number of bytes to write (length of "Hello, World\n")
    syscall                # invoke the syscall

    # exit(0)
    mov $60, %rax          # syscall number for exit (SYS_exit) is 60
    xor %rdi, %rdi         # exit status 0 (success)
    syscall                # invoke the syscall

.section .rodata          # read-only data section
message: .ascii "Hello, World\n"
```

## Is syscall very slow?

We all agree that syscall is slower compared to regular procedure call, but how
slow is it? There is no consistent answer online. Some people believe it is
every slow. Others think the overhead is not that much compared to the real
work needs to be done. The overhead is around 100 nanoseconds.

If you are concerned about syscall performance, just use `time` command. It
shows `user` and `sys` time respectively.

## Vdso
