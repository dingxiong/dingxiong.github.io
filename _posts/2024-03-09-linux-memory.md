---
layout: post
title: Linux -- Memory
date: 2024-03-09 21:11 -0800
categories: [os, linux]
tags: [linux, memory]
---

## RSS (Resident Set Size)

Resident Set is the virtual memory that is actually living inside physical
memory. It consists of three different categories: Anonymous memory pages, file
memory pages and Resident shared memory pages. See
[code](https://github.com/torvalds/linux/blob/005f6f34bd47eaa61d939a2727fc648e687b84c1/include/linux/mm_types_task.h#L28)
and
[code](https://github.com/torvalds/linux/blob/005f6f34bd47eaa61d939a2727fc648e687b84c1/include/linux/mm.h#L2613).
Note, swap entries do not count for RSS because it resides in disk.

Anonymous page refers to a memory mapping that does not have a corresponding
file on disk. This type of memory mapping is commonly used for dynamically
allocated memory, such as memory allocated with malloc in C or new in C++. The
contents of the anonymous page are typically used for data structures, stack
space, and heap allocations in user-space programs. `mmap` has a flag
parameter. One of the flags is `MAP_ANONYMOUS`. `malloc` calls `mmap` with this
flag to allocate Anonymous memory pages.

```
# cat /proc/1/status
Name:   fab
Umask:  0022
State:  S (sleeping)
Tgid:   1
Ngid:   0
Pid:    1
PPid:   0
TracerPid:      0
Uid:    0       0       0       0
Gid:    0       0       0       0
FDSize: 64
Groups:
NStgid: 1
NSpid:  1
NSpgid: 1
NSsid:  1
VmPeak:  2687036 kB
VmSize:  2653984 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:    670888 kB
VmRSS:    670888 kB
RssAnon:          609640 kB
RssFile:           60648 kB
RssShmem:            600 kB
VmData:  1125580 kB
VmStk:       156 kB
VmExe:         8 kB
VmLib:    130520 kB
VmPTE:      1732 kB
VmSwap:        0 kB
HugetlbPages:          0 kB
CoreDumping:    0
THP_enabled:    1
Threads:        28
SigQ:   0/30446
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000000000
SigIgn: 0000000001001000
SigCgt: 0000000180004203
CapInh: 0000000000000000
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
NoNewPrivs:     0
Seccomp:        0
Speculation_Store_Bypass:       vulnerable
Cpus_allowed:   ff
Cpus_allowed_list:      0-7
Mems_allowed:   00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000001
Mems_allowed_list:      0
voluntary_ctxt_switches:        306651
nonvoluntary_ctxt_switches:     9989
```

2. Explanation of /proc/$pid/status
3. Interpretation of /pro/$pid/maps
4. How does OOM killer work?
