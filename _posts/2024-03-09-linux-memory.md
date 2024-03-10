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

What is heap? TODO: detailed code study: malloc -> mmap

There are many tools to obtain memory information of a process in Linux. One is
`/proc/$pid/status`. See below example output.

```
# cat /proc/1/status
...
VmPeak:  2658284 kB
VmSize:  2658284 kB
VmLck:         0 kB
VmPin:         0 kB
VmHWM:    676124 kB
VmRSS:    676124 kB
RssAnon:          614340 kB
RssFile:           61184 kB
RssShmem:            600 kB
VmData:  1129960 kB
VmStk:       156 kB
VmExe:         8 kB
VmLib:    130520 kB
VmPTE:      1748 kB
VmSwap:        0 kB
HugetlbPages:          0 kB
...
```

The meaning of key names is manifest with one exception: `VmHWM` means
hight-water RSS usage, i.e., peak RSS. Also, not
`VmRSS = RssAnon + RssFile + RssShmem`. See
[source code](https://github.com/torvalds/linux/blob/005f6f34bd47eaa61d939a2727fc648e687b84c1/fs/proc/task_mmu.c#L64)
for where these numbers are from.

Another tool is `/proc/$pid/maps` or the `pmap` command.

```
# pmap -x 1 | head
Address           Kbytes     RSS   Dirty Mode  Mapping
0000563d95331000       4       4       0 r---- python3.11
0000563d95332000       4       4       0 r-x-- python3.11
0000563d95333000       4       0       0 r---- python3.11
0000563d95334000       4       4       4 r---- python3.11
0000563d95335000       4       4       4 rw--- python3.11
0000563d96c91000  161648  161612  161612 rw---   [ anon ]
00007f56cc000000     484     484     484 rw---   [ anon ]
00007f56cc079000   65052       0       0 -----   [ anon ]
```

`pmap` shows the start address, the virtual memory size (Kbytes), RSS size,
etc. We can sum up the anonymous page size in the output:

```
# pmap -x 1 | grep anon | awk '{s+=$3} END {print s}'
608380
```

You notice that the sum of `pmap` output is slightly different from
`/proc/$pid/status`. This is because the latter is just an approximation.
Quoted from [man proc](https://man7.org/linux/man-pages/man5/proc.5.html):

> Resident Set Size: number of pages the process has in real memory. This is
> just the pages which count toward text, data, or stack space. This does not
> include pages which have not been demand-loaded in, or which are swapped out.
> This value is inaccurate; see /proc/pid/statm below.

Third tool is `/proc/$pid/smaps`. This is just a detailed version of
`/proc/$pid/maps`.

4. How does OOM killer work?
