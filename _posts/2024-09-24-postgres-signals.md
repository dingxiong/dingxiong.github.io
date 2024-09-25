---
layout: post
title: Postgres -- Signals
date: 2024-09-24 14:40 -0700
categories: [database, postgres]
tags: [postgres]
---

## SIGALRM and Timeouts

If you have done some timer things in Linux, you must be familiar with the
concept of `Interval Timer`. If not, please check out the
[man page](https://man7.org/linux/man-pages/man2/setitimer.2.html). Below
example illustrates how to use it.

```c
#include <stdio.h>
#include <signal.h>
#include <sys/time.h>
#include <unistd.h>

static int signal_recv_count;

void sigalrm_handler(int signum)
{
  printf("SIGALRM received, count :%d\n", signal_recv_count);
  signal_recv_count++;
}

int main()
{
  struct itimerval timer={0};
  timer.it_value.tv_sec = 5;
  timer.it_interval.tv_sec = 1;

  printf("start ...\n"); fflush(stdout);
  signal(SIGALRM, &sigalrm_handler);
  setitimer(ITIMER_REAL, &timer, NULL);

  for (int i = 0; i < 10; i++) sleep(10);
}
```

Output.

```
$ ./a.out
start ...
SIGALRM received, count :0
SIGALRM received, count :1
SIGALRM received, count :2
SIGALRM received, count :3
SIGALRM received, count :4
SIGALRM received, count :5
SIGALRM received, count :6
SIGALRM received, count :7
SIGALRM received, count :8
SIGALRM received, count :9
```

When a process exits, Linux cleans up its
[itimers](https://github.com/torvalds/linux/blob/005f6f34bd47eaa61d939a2727fc648e687b84c1/kernel/time/posix-timers.c#L1093)
as well.

This is exactly how Postgres implements all sorts of
[timeouts](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/utils/timeout.h#L23).  
Interval timer relevant code is inside a single file
[timeout.c](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/misc/timeout.c#L210)
especially function `schedule_alarm`.

Take `statement_timeout` as an example. Quote from Postgres official
documentation

> statement_timeout (integer)
>
> Abort any statement that takes more than the specified amount of time. The
> timeout is measured from the time a command arrives at the server until it is
> completed by the server.

For a simple query, most logic is inside function `exec_simple_query`. You can
see that it starts
[a transaction](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/tcop/postgres.c#L1143)
before parsing the query string, and
[finishes this transaction](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/tcop/postgres.c#L1299)
after query is finished. Function `start_xact_command` calls function
`enable_statement_timeout()` which starts the statement_timeout timer.
`finish_xact_command` calls function `disable_statement_timeout` which stops
and removes the statement_timeout timer. Any time used between them are
counted!

From the blog post "Postgres -- Cliens", we know that sending result tuples to
the client is also a part of this transaction and `printtup` blocks if the
client cannot `recv` socket packets timely. By the way, the timer measures
[wall clock](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/misc/timeout.c#L339).
This creates an awkward situation. In languages that support coroutine such as
Python, when the Postgres client is implemented using coroutines, the runtime
may switch to other bad or cpu-intensive coroutines which blocks the Postgres
socket from doing `recv`. Then the query times out. The query itself is not
slow though.
