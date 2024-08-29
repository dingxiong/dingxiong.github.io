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
