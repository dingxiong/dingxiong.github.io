---
layout: post
title: Linux -- Getting started
date: 2024-03-09 23:07 -0800
categories: [os, linux]
tags: [linux, memory]
---

It is so hard to compile Linux kernel on my Macbook M1. Since I only need the
compilation database, I decided to build it in a Linux box, and then copy the
the file `compile_commands.json` out.

```bash
cd ~/code/linux

# install dependencies
sudo apt-get install flex libelf-dev bc bear liblz4-tool

# Disable certificates
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS

# enable cgroup
scripts/config --enable CONFIG_CGROUPS
scripts/config --enable CONFIG_RESOURCE_COUNTERS
scripts/config --enable CONFIG_MEMCG
scripts/config --enable CONFIG_MEMCG_SWAP
scripts/config --enable CONFIG_MEMCG_KMEM

# unselect all options
make menuconfig

# build
bear -- make -j4

# copy it out
scp ...

# fix the commands
sed -i 's;home/admin;Users/xiongding;g' compile_commands.json
```

It only took a few minutes because I disabled all options at the menu config
step. Now, `clangd` jumps perfectly!

To avoid compilation again in the future, I uploaded the compilation database
file to
[gist](https://gist.github.com/dingxiong/c4c02343cf68382f0fc83b505c61443d).

## ELF

Quote from ChatGPT:

> An ELF (Executable and Linkable Format) file is a binary file format used by
> most Unix-based operating systems for executable files, object code, shared
> libraries, and core dumps. It provides a standard way of organizing
> executable code and data, as well as facilitating the loading and execution
> of programs.

### Sections

Example sections:

```
# readelf -S /bin/ls -W
There are 30 section headers, starting at offset 0x23768:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        00000000000002a8 0002a8 00001c 00   A  0   0  1
  [ 2] .note.gnu.build-id NOTE            00000000000002c4 0002c4 000024 00   A  0   0  4
  [ 3] .note.ABI-tag     NOTE            00000000000002e8 0002e8 000020 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        0000000000000308 000308 0000ac 00   A  5   0  8
  [ 5] .dynsym           DYNSYM          00000000000003b8 0003b8 000c00 18   A  6   1  8
  [ 6] .dynstr           STRTAB          0000000000000fb8 000fb8 0005c5 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          000000000000157e 00157e 000100 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         0000000000001680 001680 0000a0 00   A  6   2  8
  [ 9] .rela.dyn         RELA            0000000000001720 001720 001440 18   A  5   0  8
  [10] .rela.plt         RELA            0000000000002b60 002b60 0009d8 18  AI  5  24  8
  [11] .init             PROGBITS        0000000000004000 004000 000017 00  AX  0   0  4
  [12] .plt              PROGBITS        0000000000004020 004020 0006a0 10  AX  0   0 16
  [13] .plt.got          PROGBITS        00000000000046c0 0046c0 000018 08  AX  0   0  8
  [14] .text             PROGBITS        00000000000046e0 0046e0 013abe 00  AX  0   0 16
  [15] .fini             PROGBITS        00000000000181a0 0181a0 000009 00  AX  0   0  4
  [16] .rodata           PROGBITS        0000000000019000 019000 005321 00   A  0   0 32
  [17] .eh_frame_hdr     PROGBITS        000000000001e324 01e324 000944 00   A  0   0  4
  [18] .eh_frame         PROGBITS        000000000001ec68 01ec68 0032a0 00   A  0   0  8
  [19] .init_array       INIT_ARRAY      0000000000023350 022350 000008 08  WA  0   0  8
  [20] .fini_array       FINI_ARRAY      0000000000023358 022358 000008 08  WA  0   0  8
  [21] .data.rel.ro      PROGBITS        0000000000023360 022360 000a78 00  WA  0   0 32
  [22] .dynamic          DYNAMIC         0000000000023dd8 022dd8 0001f0 10  WA  6   0  8
  [23] .got              PROGBITS        0000000000023fc8 022fc8 000038 08  WA  0   0  8
  [24] .got.plt          PROGBITS        0000000000024000 023000 000360 08  WA  0   0  8
  [25] .data             PROGBITS        0000000000024360 023360 000268 00  WA  0   0 32
  [26] .bss              NOBITS          00000000000245e0 0235c8 0012d8 00  WA  0   0 32
  [27] .gnu_debugaltlink PROGBITS        0000000000000000 0235c8 000049 00      0   0  1
  [28] .gnu_debuglink    PROGBITS        0000000000000000 023614 000034 00      0   0  4
  [29] .shstrtab         STRTAB          0000000000000000 023648 00011c 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

The first section is always a dummy section.

.shstrtab: Section header names table.

.interp: Interp section stores the dynamic linker path.

**.dynamic**

`.dynamic` section is used in dynamic linking. Check out
https://refspecs.linuxbase.org/elf/gabi4+/ch5.dynamic.html for the meaning of
all these tags.

```
root@xiong-test:~/mold/build# readelf -d mold

Dynamic section at offset 0x1320530 contains 33 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libz.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libcrypto.so.3]
 0x0000000000000001 (NEEDED)             Shared library: [libstdc++.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [ld-linux-x86-64.so.2]
 0x000000000000000c (INIT)               0xf0000
 0x000000000000000d (FINI)               0x1193864
 0x0000000000000019 (INIT_ARRAY)         0x12c7e80
 0x000000000000001b (INIT_ARRAYSZ)       2424 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x12c87f8
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x3d8
 0x0000000000000005 (STRTAB)             0x2558
 0x0000000000000006 (SYMTAB)             0x668
 0x000000000000000a (STRSZ)              8077 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x1321780
 0x0000000000000002 (PLTRELSZ)           5232 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0xede80
 0x0000000000000007 (RELA)               0x4a20
 0x0000000000000008 (RELASZ)             955488 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000000000001e (FLAGS)              BIND_NOW
 0x000000006ffffffb (FLAGS_1)            Flags: NOW PIE
 0x000000006ffffffe (VERNEED)            0x4780
 0x000000006fffffff (VERNEEDNUM)         7
 0x000000006ffffff0 (VERSYM)             0x44e6
 0x000000006ffffff9 (RELACOUNT)          34038
 0x0000000000000000 (NULL)               0x0
```

#### GOT (global offset table) and PLT (procedure linkage table)

I found this
[answer](https://reverseengineering.stackexchange.com/a/20214/43649) expresses
the goal and motivation behind GOT and PLT best. Also, follow
[this post](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html)
to see the assembly details.
