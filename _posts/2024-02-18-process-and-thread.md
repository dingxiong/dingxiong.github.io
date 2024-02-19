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

## Thread

Let's talk about Linux first. When you search `pthread_create` in glibc repo,
you will see two implementations. One is under `htl` folder, the other is under
`nptl` folder. These two names represent LinuxThreads and Native POSIX Thread
Library respectively. This
[paper](http://comet.lehman.cuny.edu/jung/cmp426697/NPTL.pdf) is a must read to
understand why NPTL replaces LinuxThreads.

What are the differences between a process and a thread? First, we need to be
explicit about what layer we are talking about: is it in the user land or in
the kernel? In the user land, the distinction is clear. Each process has its
own distinct PID, and its own virtual memory space. On the other hand, threads
in the same group share the same PID and memory space. In the kernel, there is
neither process nor thread. The only concept is called task. Tasks themselves
can share nothing, something, or everything.

The threading model in both LinuxThreads and NPTL is 1:1 mapping, namely, each
process or thread in the user land is mapped to a task in the kernel. To create
a new process, we use system call `fork` which creates a new kernel task that
shares nothing with its calling task. To create a new thread, we use system
call `clone` with flag `CLONE_PARENT` and `CLONE_THREAD` set, which means the
new task will share the same PID, PPID, and TGID of the the calling task.
Actually, `fork` is implemented as a wrapper on top of `clone`. More detailed
discussion can be found [here](https://unix.stackexchange.com/a/364834/159849).

So let's take a look at the source code of
[pthread_create](https://github.com/bminor/glibc/blob/88b771ab5e1169e746dbf4a990d90cffc5fa54ea/nptl/pthread_create.c#L624)
quickly. The signature is below.

```c
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg) {
    ...
}
```

First, note that it takes a start function and an argument. This is different
from a process. After `fork`, the child process continues to run the rest code,
but a new thread will run the provided function until it finishes and the
thread exits.

Second, the data structure `__pthread_attr_t` defines the properties/flags of
the new thread. If is defined as a union
[here](https://github.com/bminor/glibc/blob/1d44530a5be2442e064baa48139adc9fdfb1fc6b/sysdeps/nptl/bits/pthreadtypes.h#L56).
It is quite wired! It is just a byte array. The `int` member is only for align
purpose and won't be used. This shouldn't be the real definition because
`__pthread_attr_t` supposes to carry flags that define what to share between
the calling thread and new thread. The actual definition is
[here](https://github.com/bminor/glibc/blob/1d44530a5be2442e064baa48139adc9fdfb1fc6b/sysdeps/nptl/internaltypes.h#L26)
copied below

```c
struct pthread_attr
{
  /* Scheduler parameters and priority.  */
  struct sched_param schedparam;
  int schedpolicy;
  /* Various flags like detachstate, scope, etc.  */
  int flags;
  /* Size of guard area.  */
  size_t guardsize;
  /* Stack handling.  */
  void *stackaddr;
  size_t stacksize;

  /* Allocated via a call to __pthread_attr_extension once needed.  */
  struct pthread_attr_extension *extension;
  void *unused;
};
```

`__pthread_attr_t` is always statically casted to `struct pthread_attr`. Why
design it this way? glibc runs in the user land, `pthread_attr` is the data
structure used in the user land. We need to do a sys call `clone` and we need
to pass bytes to kernel which is `__pthread_attr_t`. So it is the same thing in
two representations.

Third, `__pthread_attr_t` has two members `stackaddr` and `stacksize`. These
two members defines the stack memory space of the new thread. A few lines below
is

```c
  int err = allocate_stack (iattr, &pd, &stackaddr, &stacksize);
```

The stack space of the new thread is allocated in the user land instead of in
the kernel! Most time, the caller leaves these two parameters unspecified, so
the default stack size is used.

```bash
$ ulimit -a | grep stack
stack size                  (kbytes, -s) 10240
```

Above example shows that the default stack size is 10MB, which is a lot, right?
Suppose the total memory is 16GB, then we can have at most 1600 threads? No.
The calculation is way off. The thread's stack size is just the size in the
virtual memory space. The stack may or may not be mapped to physical memory
depending on whether the space is used or not. If the stack usage is under a
page size, for example 64KB, then only one page is allocated in physical
memory. In reality, there is no problem to run millions of threads in Linux.
For the stack size calculation, please see Ulrich's blog post
[Thread Numbers and Stacks](https://www.akkadia.org/drepper/thread-number-stacks.html).
Meanwhile, I personally find
[this post](https://linuxgazette.net/112/krishnakumar.html) quite enlightening
as well.

### Inter thread communication

TODO:

1. read https://www.akkadia.org/drepper/futex.pdf
2. read https://www.akkadia.org/drepper/tls.pdf

## Relearn ps

ps is the most used command in Linux. I am learning new things about it from
time to time. I definitely cannot know all its usage.

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
