---
layout: post
title: Linux -- Cgroup
date: 2025-01-12 11:43 -0800
categories: [os, linux]
tags: [linux, cgroup]
---

## Memory Resource Controller

When you exec into a k8s pod, you will find a lot of files under
`/sys/fs/cgroup/memory`.

```
root@kafka-all-in-one-consumer-7567f685c4-8wv8x:/sys/fs/cgroup/memory# ls -l
total 0
-rw-r--r-- 1 root root 0 Jun  1 16:00 cgroup.clone_children
--w--w--w- 1 root root 0 Jun  1 15:38 cgroup.event_control
-rw-r--r-- 1 root root 0 Jun  1 16:09 cgroup.procs
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.failcnt
--w------- 1 root root 0 Jun  1 16:09 memory.force_empty
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.failcnt
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.limit_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.max_usage_in_bytes
-r--r--r-- 1 root root 0 Jun  1 16:09 memory.kmem.slabinfo
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.tcp.failcnt
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.tcp.limit_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.tcp.max_usage_in_bytes
-r--r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.tcp.usage_in_bytes
-r--r--r-- 1 root root 0 Jun  1 16:00 memory.kmem.usage_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.limit_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.max_usage_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.memsw.failcnt
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.memsw.limit_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.memsw.max_usage_in_bytes
-r--r--r-- 1 root root 0 Jun  1 16:00 memory.memsw.usage_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:09 memory.move_charge_at_immigrate
-r--r--r-- 1 root root 0 Jun  1 16:00 memory.numa_stat
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.oom_control
---------- 1 root root 0 Jun  1 16:09 memory.pressure_level
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.soft_limit_in_bytes
-r--r--r-- 1 root root 0 Jun  1 16:00 memory.stat
-rw-r--r-- 1 root root 0 Jun  1 16:09 memory.swappiness
-r--r--r-- 1 root root 0 Jun  1 16:00 memory.usage_in_bytes
-rw-r--r-- 1 root root 0 Jun  1 16:00 memory.use_hierarchy
-rw-r--r-- 1 root root 0 Jun  1 16:09 notify_on_release
-rw-r--r-- 1 root root 0 Jun  1 16:06 tasks
```

See explanation for these files in
[Linux kernel documentation](https://docs.kernel.org/admin-guide/cgroup-v1/memory.html).
I want to call out the `tasks` file. It stores the task ids, i.e., thread ids
belonging to this cgroup. When oom killer needs to find a process to kill, it
basically iterate all task ids of this cgroup
[[code]](https://github.com/torvalds/linux/blob/929ed21dfdb6ee94391db51c9eedb63314ef6847/mm/oom_kill.c#L364).
Because threads of the same thread group share memory. Killing one thread is
the same of killing the corresponding process.
