---
title: python asyncio - 2. Subprocess
date: 2023-11-25 23:36 -0800
categories: [programming-language, python]
tags: [asyncio, subprocess]
---

`asyncio.create_subprocess_exec()` and `asyncio.create_subprocess_shell()` are
the async counterparty of `subprocess.Popen` in the multi-process world.

For each subprocess, it creates a thread to
[watch it](https://github.com/python/cpython/blob/595ef03c7ceb25e63878e2a9231e1b54f9f0d125/Lib/asyncio/unix_events.py#L1399).
Basically, this watching thread will block at `os.watchpid()` until the
subprocess exits. When it happens, the watching thread calls a callback, and
this callback just call `asyncio.Future.set_result()`.
