---
layout: post
title: Process and thread
date: 2024-02-18 14:19 -0800
categories: [os]
tags: [os, linux, process, thread]
---

## Process

### IPC (Interprocess communication)

`wait` system call blocks a parent process until a child process terminates. If
not called, the child process will be a zombie process. How could we avoid
blocking parent process and also reap the terminated child process? The answer
is `SIGCHLD`. We can install a signal handler for `SIGCHLD` and call `wait` in
this signal handler, so parent only waits when child indeed terminates. See
http://web.stanford.edu/~hhli/CS110Notes/CS110NotesCollection/Topic%202%20Multiprocessing%20(4).html

#### pipe

I created a short program as below.

```python
def main():
    [rf, wf] = os.pipe()
    for fd in [rf, wf]:
        flags = fcntl.fcntl(fd, fcntl.F_GETFL) | os.O_NONBLOCK
        fcntl.fcntl(fd, fcntl.F_SETFL, flags)
        flags = fcntl.fcntl(fd, fcntl.F_GETFD)
        flags |= fcntl.FD_CLOEXEC
        fcntl.fcntl(fd, fcntl.F_SETFD, flags)

    pid = os.fork()
    if pid:
        # parent
        bs = os.read(rf, 1)
        print(bs)
    else:
        # child
        os.write(wf, b'a')
```

However, running it produced below error.

```
    bs = os.read(rf, 1)
         ^^^^^^^^^^^^^^
BlockingIOError: [Errno 35] Resource temporarily unavailable
```

Doc page of `os.read` does not mention this error. But I found the doc page of
[io.BufferedIOBase.read](https://docs.python.org/3/library/io.html#io.BufferedIOBase.read)
says

> A BlockingIOError is raised if the underlying raw stream is in non
> blocking-mode, and has no data available at the moment.

Then I went to
[Cpython source code](https://github.com/python/cpython/blob/f3909b8bc83675b7ab093dbc558e677558d8e718/Objects/exceptions.c#L3655).
Ah. `EAGAIN` is classified as a BlockingIOError. `man 2 read` gives more
details about it.

We can use `select` to fix this issue. See the full code in `pipe_example.py`
in the tutorial folder.

##### Concurrent write

If multiple processes are writing to the same pipe, then the messages may get
interleaved. However, when the message length is less than PIPE_BUF, then the
message can be written atomically. POSIX.1 requires PIPE_BUF to be at least 512
bytes. See `man 7 pipe` for more details.

## Relearn ps

ps is the most used command in Linux. I am learning new things about it from
time to time. I definitely cannot know all its usage.

### Threads

Quote from ps man page

```
 H      Show threads as if they were processes.
 -L     Show threads, possibly with LWP and NLWP columns.
 -m     Show threads after processes.
 -T     Show threads, possibly with SPID column.
```

Let's see a few examples

```
# ps -efL
UID        PID  PPID   LWP  C NLWP STIME TTY          TIME CMD
root         1     0     1  1   15 15:38 ?        00:00:16 kafka consumer controller
root         1     0    16  0   15 15:38 ?        00:00:00 kafka consumer controller
root         1     0    17  0   15 15:38 ?        00:00:00 kafka consumer controller

# ps -efT
UID        PID  SPID  PPID  C STIME TTY          TIME CMD
root         1     1     0  1 15:38 ?        00:00:16 kafka consumer controller
root         1    16     0  0 15:38 ?        00:00:00 kafka consumer controller
root         1    17     0  0 15:38 ?        00:00:00 kafka consumer controller
```

Here, `LWP` stands for `Light Weighted process`, i.e., thread. `NLWP` means the
number of threads in this thread group. `SPID` is an alias of `LWP`. It
possibly stands for shared PID. You can checkout ps's man page to read more
about these jargons.

## Mics

### read vs pread

`read` will read from current offset and advance offset. `pread` takes a
`offset` parameter and does not change current offset of the file.

## Thread

Let's talk about Linux first. When you search `pthread_create` in glibc repo,
you will see two implementations. One is under `htl` folder, the other is under
`nptl` folder. These two names represent LinuxThreads and Native POSIX Thread
Library respectively. This
[paper](http://comet.lehman.cuny.edu/jung/cmp426697/NPTL.pdf) is a must read to
understand why NPTL replaces LinuxThreads.

Difference between fork and clone.

TODO:

1. what is thread stack? where it is and what is inside?
2. a section about futex. so I can fully understand how mutex is implemented.
