---
title: Mysql atomicity
date: 2023-12-09 00:10 -0800
categories: [database, mysql]
tags: [mysql]
---

## File system: block vs sector

Sector is the smallest unit a disk can read/write, so writing a sector is
atomic. Sector size is usually 512 bytes. On the other hand, block size refers
to the allocation size the file system uses. A block consists of multiple of
sectors.

## Is writing to a fs block atomic?

I do not find an authoritative source about fs write atomicity. But Linux
`write/pwrite` system call is not atomic. Quote from
[man pwrite(3)](https://linux.die.net/man/3/pwrite),

> Applications need to know how large a write request can be expected to be
> performed atomically. This maximum is called {PIPE_BUF}

Usually `PIPE_BUF` is 512 bytes. We learned this parameter in Linux pipes.
Right?

Paper
[All File Systems are Not Created Equal](https://blog.acolyer.org/2016/02/11/fs-not-equal/)
compares atomic properties of different fs.

## Innodb redolog atomicity

A good article on this topic from alibaba cloud:
[An In-Depth Analysis of REDO Logs in InnoDB](https://www.alibabacloud.com/blog/an-in-depth-analysis-of-redo-logs-in-innodb_598965).

Redo logs are first created at
[mtr_t.m_log field](https://github.com/mysql/mysql-server/blob/a246bad76b9271cb4333634e954040a970222e0a/storage/innobase/include/mtr0mtr.h#L188),
which is a linked list of byte blocks. Here, `mtr` stands for mini-transaction.
This heap space is called InnoDB Log Buffer. Being a linked list means a single
page update may contain multiple redo logs. Each byte block has fixed size
[512 bytes](https://github.com/mysql/mysql-server/blob/a246bad76b9271cb4333634e954040a970222e0a/storage/innobase/include/dyn0types.h#L43-L44).

During `mtr_t.commit()`, redo logs are written to page cache by
[pwrite](https://github.com/mysql/mysql-server/blob/a246bad76b9271cb4333634e954040a970222e0a/storage/innobase/os/os0file.cc#L2047)
system calls. The write block size is also
[512 bytes](https://github.com/mysql/mysql-server/blob/a246bad76b9271cb4333634e954040a970222e0a/storage/innobase/include/os0file.h#L191-L192).
Not one redo log may occupy multiple file log blocks, so each log has a Header
and Trailer block to denote the start and end of a redo log. During recovery
after power off, the header and trailer block are used to decide whether a redo
log is complete.

Before `mtr_t.commit()` returns, it calls `fsync` to persist page cache changes
to disk.
