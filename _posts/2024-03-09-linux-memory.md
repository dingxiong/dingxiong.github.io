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
`/proc/$pid/maps`. Example output of `/proc/$pid/smpas` is

```
...
7f8a99522000-7f8a99677000 r-xp 00026000 103:01 153103435                 /usr/lib/x86_64-linux-gnu/libc.so.6
Size:               1364 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                 536 kB
Pss:                 205 kB
Shared_Clean:        468 kB
Shared_Dirty:          0 kB
Private_Clean:        68 kB
Private_Dirty:         0 kB
Referenced:          536 kB
Anonymous:             0 kB
--
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:            0
ProtectionKey:         0
VmFlags: rd mr mw me sd
...
```

One interesting field is `Pss` (Proportional Set Size). It is used to determine
the amount of memory that is actually being used by a process, taking into
account the shared memory that may be used by multiple processes. PSS is
calculated by dividing the shared memory by the number of processes sharing
that memory and adding it to the private memory (memory that is not shared).

### OOM killer

Most OOM logic resides inside `oom_killer.c`. The entry point is function
[out_of_memory](https://github.com/torvalds/linux/blob/005f6f34bd47eaa61d939a2727fc648e687b84c1/mm/oom_kill.c#L1104)
The call stack to OOM is

```
__alloc_pages
    |-->__alloc_pages_nodemask
       |--> __alloc_pages_slowpath
           |--> __alloc_pages_may_oom
              | --> out_of_memory
```

OOM killer deals with two different cases: normal Linux processes and Linux
cgroups. If a namespace is about to violate its memory limitation, OOM killer
will pick one process in the cgroup to kill.

In practice, I found this
[log](https://github.com/torvalds/linux/blob/005f6f34bd47eaa61d939a2727fc648e687b84c1/mm/oom_kill.c#L946)
most useful when reading syslogs. One example below

```
Memory cgroup out of memory: Killed process 19513 (fab) total-vm:7196300kB, anon-rss:4173736kB, file-rss:58444kB, shmem-rss:600kB, UID:0 pgtables:8824kB oom_score_adj:904
```

One side note about logs from containerd. The logs come from
[code](https://github.com/containerd/containerd/blob/9fc0b64bc40b0a92bae192737869131c978898df/internal/cri/server/events.go#L310).
One example below.

```
TaskOOM event &TaskOOM{ContainerID:a38973e2086019e6411254a3b9e2b8885929f005f4c814414739227673df7e56,XXX_unrecognized:[],}
```

TODO: figure out the code path: malloc -> sys_brk -> \_\_alloc_pages

### cgroup memory notifier

At current company, I desperately need a way to add more debugging info to OOM
events. Eng team is complaining that they do not know where to investigate when
OOM happens. The
[cgroup memory manual](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)
says `memory.oom_control` can be used to register a notifier, but I kind of
suspect it won't be effective for us because this mechanism is totally async.
When the notifier receives the event, the target process is probably terminated
already. So I guess a better way is setting a threshold notifier: use
`cgroup.event_control` with a threshold say `memory.limit_in_bytes * 0.95`.

There is a good C example online. See
[code](https://github.com/matteobertozzi/blog-code/blob/master/cgroup-mem-threshold/cgroup-mem-threshold.c).
I translated it to python as below

```python
import os
import sys
from pathlib import Path
import selectors

def main(threshold_bytes: int):
    usage_fd = -1
    event_fd = -1
    try:
        base_dir = Path("/sys/fs/cgroup/memory")

        usage_fd = os.open(base_dir / "memory.usage_in_bytes", os.O_RDONLY)
        event_fd = os.eventfd(0, os.EFD_NONBLOCK | os.EFD_CLOEXEC)
        print(f"event_fd {event_fd}  usage_fd {usage_fd}")

        event = f"{event_fd} {usage_fd} {threshold_bytes}"
        with open(base_dir / "cgroup.event_control", "w") as f:
            f.write(event)

        selector = selectors.DefaultSelector()
        selector.register(event_fd, selectors.EVENT_READ)
        while True:
            events = selector.select()
            for key, mask in events:
                bs = os.read(event_fd, 8)
                val = int.from_bytes(bs, sys.byteorder)
                print("Notfication", key, val, mask)

    finally:
        if usage_fd > 0:
            os.close(usage_fd)
        if event_fd > 0:
            os.close(event_fd)

if __name__ == "__main__":
    main(threshold_bytes=2921592064)
```

When running it inside a k8s pod, I immediately received an error saying file
`cgroup.event_control` is read-only! Then I found this
[issue](https://github.com/kubernetes/kubernetes/issues/121190). Basically, we
need to create the pod with below config.

```
securityContext:
  privileged: true
```

After getting the basic mechanism working, the next step is to sending signals
to relevant processes when event is triggered, and all processes should set a
signal handler to dump stacktrace.
