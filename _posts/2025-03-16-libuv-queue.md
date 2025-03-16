---
layout: post
title: Libuv -- Queue
date: 2025-03-16 13:00 -0700
categories: [programming-language, c]
tags: [libuv]
---

Libuv implements a double-linked closed list using macros. The first time I
read it, I feel a little lost because the queue exists both in `uv__io_t` and
`loop`. Then I read
[Bodo Kaiser's explanation](https://gist.github.com/bodokaiser/5657156). Ah! It
suddenly makes sense. I realize I missed the following points the first time I
read it.

1. The queue definition `typedef void *QUEUE[2];` actually defines the queue
   node.
2. The first node in the queue is a dummy node. `QUEUE_NEXT(q)` is the real
   first node in the queue. That is also why empty check is defines as
   ```
   #define QUEUE_EMPTY(q)                                                        \
     ((const QUEUE *) (q) == (const QUEUE *) QUEUE_NEXT(q))
   ```
3. Queue node's data is retrieved by `offsetof` function. This design is quite
   different from OOP, or even opposite. In OOP, your queue data should be a
   member of a node class. While, here we embed queue node as a member of the
   data struct. Actually, after thinking for a while, these two approaches seem
   to be the same thing. In OOP, the natural definition would be
   ```cpp
   template<typename T>
   class Node<T> {
     T data;
   };
   ```
   Here, it is more like
   ```cpp
   class AbstractNode {
     virtual void* getData();
   };
   using Node = *AbstractNode[2];
   ```
   The later is more flexible, it does not need to embed data as part of the
   node. We are just used to the OOP approach though. Also, this trick is used
   at quite a few other places in libuv.

## Comments for a few operations

`QUEUE_MOVE(h, n)`: All elements after dummy node `h` will be moved to dummy
node `n`. After moving, `h` will be empty, and `n` will have all nodes moved
from `h`.

## uv\_\_io_feed

Very confused by this function.

1. Why it checks the queue is empty? If empty, nothing is inserted to the tail
   except the dummy mode.
2. Why add the dummy node? will it break the logic somewhere? This dummy node
   actually has some callback `uv__io_cb`

## Pending queue

`pending_queue` is defined in two places:

1. [uv\_\_io_s](https://github.com/libuv/libuv/blob/e7ecd116e0e013bf8c01d66505ced4ce1b58c005/include/uv/unix.h#L95)
2. [uv_loop_s](https://github.com/libuv/libuv/blob/e7ecd116e0e013bf8c01d66505ced4ce1b58c005/include/uv/unix.h#L223)

They actually have different semantics. As said before, `QUEUE` type is
actually queue node type. The `pending_queue` in `uv__io_s` is a normal node,
namely, non-dummy node. So this node should have callback associated. In
contrary, the `pending_queue` in `uv_loop_s` is a dummy node.

There is only one place that `pending_queue` in `uv__io_t` is added to
`uv_loop_s`. That is `uv__io_feed`.

```c
void uv__io_feed(uv_loop_t* loop, uv__io_t* w) {
  if (QUEUE_EMPTY(&w->pending_queue))
    QUEUE_INSERT_TAIL(&loop->pending_queue, &w->pending_queue);
}
```

Note that before appending `&w->pending_queue` it checks whether it is empty or
not. This makes sense as `uv__io_t` represents the state of a single IO source,
like a socket. There should only be one callback associated.

## watcher queue
