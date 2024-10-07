---
layout: post
title: Postgres -- Clients
date: 2024-08-26 07:42 -0700
categories: [database, postgres]
tags: [postgres]
---

Postgres official documentation
[Chapter 55. Frontend/Backend Protocol](https://www.postgresql.org/docs/16/protocol.html)
covers all the details about the client-sever binary protocol, which is a based
on top of TCP.

> All communication is through a stream of messages. The first byte of a
> message identifies the message type, and the next four bytes specify the
> length of the rest of the message (this length count includes itself, but not
> the message-type byte). The remaining contents of the message are determined
> by the message type. For historical reasons, the very first message sent by
> the client (the startup message) has no initial message-type byte.

## Java

[pgjdbc](https://github.com/pgjdbc/pgjdbc/tree/master)

See [doc](https://github.com/pgjdbc/pgjdbc/blob/master/CONTRIBUTING.md) for how
to build.

```
./gradlew build -x test
```

## When is the result returned to the client?

When does the data come back to you when sending a select query to a Postgres
database? It definitely boils down to socket programming, but I want to know
when and how it happens. I added a hook inside function `socket_putmessage` to
print out stack traces. Below is what I captured during a simple select query.

```
0   0x0000000102a87900 socket_putmessage + 96
1   0x0000000102881eb4 SendRowDescriptionMessage + 780
2   0x0000000102a38478 standard_ExecutorRun + 188
3   0x0000000102c105f4 PortalRunSelect + 232
4   0x0000000102c101fc PortalRun + 512
5   0x0000000102c0f0a8 exec_simple_query + 1360
6   0x0000000102c0cd4c PostgresMain + 3592

0   0x0000000102a87900 socket_putmessage + 96
1   0x0000000102881a24 printtup + 724
2   0x0000000102a3854c standard_ExecutorRun + 400
3   0x0000000102c105f4 PortalRunSelect + 232
4   0x0000000102c101fc PortalRun + 512
5   0x0000000102c0f0a8 exec_simple_query + 1360
6   0x0000000102c0cd4c PostgresMain + 3592

0   0x0000000102a87900 socket_putmessage + 96
1   0x0000000102c09634 EndCommand + 88
2   0x0000000102c0f234 exec_simple_query + 1756
3   0x0000000102c0cd4c PostgresMain + 3592
4   0x0000000102c086c4 BackendInitialize + 0
5   0x0000000102b6d2dc PgArchShmemSize + 0

0   0x0000000102a87900 socket_putmessage + 96
1   0x0000000102a8806c pq_endmessage + 48
2   0x0000000102c09730 ReadyForQuery + 108
3   0x0000000102c0c5b4 PostgresMain + 1648
```

First, the sequence of messages to the client: `SendRowDescriptionMessage` ->
`printtup` -> `EndCommand` -> `ReadyForQuery` is expected. Nothing special
here. If the result has multiple tuples, then `printtup` will show up multiple
times. Postgres sends the result tuples to the client on the fly without
assembling them first. This also means that sending tuples to the client is
inside the same transaction as parsing, planing and execution.

A natural follow-up question is: does `printtup` block? Yes, it does! This
actually surprised me when I read about it first. Initially, I thought Postgres
server must use a non-blocking socket underneath. If the client is busy doing
other things and cannot `recv` these tuples timely, then the server can put
them else where temporarily so it can switch to do something else. It is
inefficient to block inside `printtup`. Also, it makes the transaction
unnecessarily longer, which has a negative impact on locks, MVCC, etc. However,
taking a step back, what else could the server do if the client is not
responding? Nothing. Postgres uses a process model. Each client has a dedicate
server process serving it. There is no multiplexing, so waiting/blocking is
that bad.

Let's see how `printtup` blocks. First, I get one part correct: socket is
always created in
[non-blocking mode](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/libpq/pqcomm.c#L294).
Function `pg_init` returns a `Port` object, which holds information for a
client's socket connection. The interesting thing is that this struct has a
member field called
[noblock](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/libpq/libpq-be.h#L135).
So I guess it uses field `noblock` to simulate blocking behavior of a
non-blocking socket. And this is exactly what happens underneath. The call
stack is

```
socket_putmessage
-> internal_putbytes
  -> internal_flush
    -> internal_flush_buffer
      -> secure_write
```

Function `internal_putbytes` sets `noblock` to `false` before it flushes the
send buffer. Then see
[part of secure_write code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/libpq/be-secure.c#L331).

```c
if (n < 0 && !port->noblock && (errno == EWOULDBLOCK || errno == EAGAIN))
{
  ...
  goto retry;
}
```

`EAGAIN` and `EWOULDBLOCK` are error codes only for non-blocking socket,
meaning the internal buffer is full for send, and thus it cannot proceed. In
this case, the data is still in the application's memory (the buffer you passed
to the send() function). Therefore, the above code means if port is blocking,
then it will retry sending the data to client until the client has received it.
So this behavior is similar to a blocking socket. Why does Postgres choose to
implement this behavior in the application layer instead of changing the socket
from non-blocking to blocking using `fcntl`. Because it saves one Linux system
call?

This blocking design has a big impact on the timeout behavior. See the blog
post about Postgres signals.

## Pgbouncer

```
./configure --with-openssl=/opt/homebrew/opt/openssl@3
bear -- make -j7
```
