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

## ClientRead

I see ClientRead accounts for 90% of db wait type. What is that? The place that
set this wait type is
[here](https://github.com/postgres/postgres/blob/3f9b9621766796983c37e192f73c5f8751872c18/src/backend/libpq/be-secure.c#L218).
A typical call stack is

```
1   postgres                            0x0000000104ab3034 pq_recvbuf + 16
2   postgres                            0x0000000104ab360c pq_getbytes + 124
3   postgres                            0x0000000104ab3b50 pq_getmessage + 784
4   postgres                            0x0000000104d43cf8 SocketBackend + 832
5   postgres                            0x0000000104d40650 ReadCommand + 44
6   postgres                            0x0000000104d3fbe0 PostgresMain + 2452
7   postgres                            0x0000000104d37634 BackendInitialize + 0
8   postgres                            0x0000000104c218c0 postmaster_child_launch + 324
9   postgres                            0x0000000104c28da0 BackendStartup + 296
10  postgres                            0x0000000104c26de4 ServerLoop + 372
11  postgres                            0x0000000104c25ba0 PostmasterMain + 6304
12  postgres                            0x0000000104ab78d4 main + 904
13  dyld                                0x0000000187205d54 start + 7184
```

It happens when server is waiting for client to send more data. Remember that
each message has 5 bytes header: 1 byte for type + 4 bytes for message length.
So, after decoding the header, postgres waits for the full message before it
can execute it. The period is call ClientRead.

If ClientRead is high, then it means client side is busy with sending the full
query. Does the network needs more bandwidth, or is the query too long?

## Pgbouncer

Compilation

```bash
./configure --with-openssl=/opt/homebrew/opt/openssl@3
bear -- make -j7
```

Run it

```
./pgbouncer -vvv pgbouncer.ini

# file pgbouncer.ini
[databases]
postgres = host=localhost port=5434 dbname=postgres

[pgbouncer]
listen_port = 6432
listen_addr = localhost
auth_type = md5
auth_file = userlist.txt
logfile = pgbouncer.log
pidfile = pgbouncer.pid
admin_users = xiongding

# file userlist.txt
"xiongding" "password"
```

### Connection event loop

If I ask you to write a simple TCP or HTTP server, what would you do? I would
write it like below.

```python
def run_sever():
  sock = create_socket(...)
  sock.bind(...)
  sock.listen(...)

  while True:
    client_sock, addr = sock.accept(...)
    threading.Thread(target=process, args=[client_sock]).start()

def process(client_sock):
  request = client_sock.recv(...)
  resp = get_response(request)
  client_sock.sendall(resp)
  client_socket.close()
```

It processes each request in a new thread and it is easy to understand. Now
let's come to the async world. Since C does not support coroutines natively.
Pgbouncer uses `libevent` to do asyncio. That is a hell of event loop
callbacks!

First, let's see how client connection is established. The basic stack trace is
below.

```
main -> pooler_setup -> parse_addr -> add_listen
                     -> resume_pooler
```

Inside `add_listen`, you see the socket creation, bind and listen steps. At the
end of `pooler_setup` is a call to `resume_pooler` which add `pool_accept` to
the
[event loop](https://github.com/pgbouncer/pgbouncer/blob/1dbde965e6782f6800eb4d1bb6b4d4002bfd1323/src/pooler.c#L446).
Note the event has flag `EV_PERSIST` enabled. So instead of doing the infinite
loop itself, it persist the `accept` logic to the event loop, and the callback
will be triggered whenever there is a new client coming.

So what is the accept logic? In our naive implementation. The accept logic
spawning a new thread, and then put all the business logic inside this thread.
Here, in pgbouncer/libevent, there is only a single thread, we need be smarter,
i.e., more callback hell! The call stack is below.

```
pool_accept
  -> accept_client: create PgSocket struct.
    -> sbuf_accept:
      -> sbuf_wait_for_data: add sbuf_recv_cb event to the event loop. Note, this is also a EV_PERSIST event.
        -> sbuf_recv_cb
          -> sbuf_main_loop
```

Stream buffer `sbuf` is a core concept of pgbouncer. In this accept function,
it enqueues a persist event `sbuf_wait_for_data`. You can guess this function
keeps reading data from client and forwarding to the sever. These corresponding
to this
[line](https://github.com/pgbouncer/pgbouncer/blob/1dbde965e6782f6800eb4d1bb6b4d4002bfd1323/src/sbuf.c#L1024)
and
[line](https://github.com/pgbouncer/pgbouncer/blob/1dbde965e6782f6800eb4d1bb6b4d4002bfd1323/src/sbuf.c#L1031).

A sample stack trace captured is shown below.

```
0   pgbouncer                           0x0000000104fbd120 raw_sbufio_send + 68
1   pgbouncer                           0x0000000104fbc73c sbuf_send_pending_iobuf + 132
2   pgbouncer                           0x0000000104fbd6fc sbuf_process_pending + 856
3   pgbouncer                           0x0000000104fbbc84 sbuf_main_loop + 372
4   libevent-2.1.7.dylib                0x000000010510267c event_process_active_single_queue + 944
5   libevent-2.1.7.dylib                0x00000001050ff548 event_base_loop + 952
6   pgbouncer                           0x0000000104fb07d0 main_loop_once + 36
7   pgbouncer                           0x0000000104fb01ec main + 1796
8   dyld                                0x0000000187205d54 start + 7184
```

### FAQ

#### Does pgbouncer buffer the complete query before sending to server?

Yes for prepared statement but no for for simple queries.

Let's see simple queries first. I did a quick experiment that I explicitly
split a 30 bytes packet to two packet (10 + 20) on the client side and sleep 3
seconds between them.

```
diff --git a/src/interfaces/libpq/fe-misc.c b/src/interfaces/libpq/fe-misc.c
index 928a47162d..4d721bbfc4 100644
--- a/src/interfaces/libpq/fe-misc.c
+++ b/src/interfaces/libpq/fe-misc.c
@@ -972,7 +972,15 @@ pqFlush(PGconn *conn)
                if (conn->Pfdebug)
                        fflush(conn->Pfdebug);

-               return pqSendSome(conn, conn->outCount);
+               int count = conn->outCount;
+               if (count > 10) {
+                       pqSendSome(conn, 10);
+                       count -= 10;
+                       printf("sleeping...\n"); fflush(stdout);
+                       sleep(3);
+               }
+               return pqSendSome(conn, count);
+               // return pqSendSome(conn, conn->outCount);
        }
```

The pgbouncer logs were

```
2025-09-27 08:53:41.007 PDT [35293] NOISE C-0xc91800010: postgres/xiongding@127.0.0.1:58230 read pkt='Q' len=30
2025-09-27 08:53:41.007 PDT [35293] NOISE C-0xc91800010: postgres/xiongding@127.0.0.1:58230 add_outstanding_request: added Q, still outstanding 1
2025-09-27 08:53:41.007 PDT [35293] NOISE sbuf_process_pending: loop 2
2025-09-27 08:53:41.007 PDT [35293] NOISE sbuf_process_pending: done looping
2025-09-27 08:53:41.007 PDT [35293] NOISE sbuf_send_pending_iobuf
2025-09-27 08:53:41.009 PDT [35293] NOISE safe_send(11, 10) = 10
2025-09-27 08:53:41.011 PDT [35293] NOISE sbuf_process_pending: end
2025-09-27 08:53:41.011 PDT [35293] NOISE resync(10): done=10, parse=10, recv=10
2025-09-27 08:53:44.010 PDT [35293] NOISE resync(10): done=0, parse=0, recv=0
2025-09-27 08:53:44.011 PDT [35293] NOISE safe_recv(10, 4096) = 20
2025-09-27 08:53:44.011 PDT [35293] NOISE sbuf_process_pending: start
2025-09-27 08:53:44.011 PDT [35293] NOISE sbuf_process_pending: loop 1
2025-09-27 08:53:44.011 PDT [35293] NOISE sbuf_process_pending: loop 2
2025-09-27 08:53:44.011 PDT [35293] NOISE sbuf_process_pending: done looping
2025-09-27 08:53:44.011 PDT [35293] NOISE sbuf_send_pending_iobuf
2025-09-27 08:53:44.013 PDT [35293] NOISE safe_send(11, 20) = 20
```

Here `safe_send(11, 10) = 10` means sending 10 bytes to fd 11 and 10 bytes
fully sent. See
[code](https://github.com/pgbouncer/pgbouncer/blob/83d634799383d352564e08746bfd90d5c9da481a/lib/usual/safeio.c#L80).
It sent two packets to server with an interval of three seconds! Therefore, for
simple queries, pgbouncer forwards the bytes almost as soon as it receives it.
I said "almost" because it will wait for the packet buffer (5 bytes).

For prepared statement, the relevant code is
[here](https://github.com/pgbouncer/pgbouncer/blob/1dbde965e6782f6800eb4d1bb6b4d4002bfd1323/src/client.c#L1324).
Basically, if prepared statement cache is enabled, pgbouncer will wait for the
full message body for prepared queries. This makes sense because without the
full body it could not determine whether this prepared statement exists in
cache or not.

Lastly, no matter which case it is, pgbouncer always waits for the
[full header](https://github.com/pgbouncer/pgbouncer/blob/1dbde965e6782f6800eb4d1bb6b4d4002bfd1323/src/client.c#L1547).
But 5 bytes is trivial.

I spent a few hours to figure this out. Pgbouncer should learn from Nginx.
Nginx's
[documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
makes it clear that the default behavior is buffering the entire request body.
But you can turn it off by setting `proxy_request_buffering off`.
