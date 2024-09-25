---
layout: post
title: Golang -- Channel
date: 2024-09-20 14:54 -0700
categories: [programming-language, golang]
tags: [golang]
---

Channel is usually used together with goroutines. Let's take an example from
[k8s source code](https://github.com/kubernetes/kubernetes/blob/1850794626bb995bb754d54be3328c86ee880ba5/vendor/github.com/opencontainers/runc/libcontainer/notify_linux.go#L20).
This function returns a channel for callers to consume. Inside, it spawns a
background goroutine to send messages to this channel. Note, this channel is
created outside this goroutine but closed inside it. I am quite familiar with
this pattern in Python: generator. Most time, a generator is used as a
"container" in a for-loop. This is the same for goroutines. Also, note that
errors are all handled inside this goroutine. It is golang's philosophy to deal
with error instead of popping up error.

### Receive Operator

```
x := <-ch
x, ok := <-ch
```

The implementation is
[here](https://github.com/golang/go/blob/7ba074fe43a3c1e9a35cd579520d7184d3a20d36/src/runtime/chan.go#L505)

Use space usage has `block=True`, so ignore all `!block` branches. The function
works in this sequence:

1. First, check whether the channel is nil. If it is, then this coroutine
   blocks forever. You can tell from enum `traceBlockForever`.
2. Then it check whether the channel is closed or not. Note `c.closed != 0`
   means channel is closed. When `c.closed != 0 && c.qcount == 0` namely,
   channel is closed and nothing in the buffer, it mem-resets the content of
   receiver return by `typedmemclr` and return `received = Fasle`. One the
   other hand if `c.close == 0 && c.sendq is not empty`, namely, channel is
   active and there is also a sender waiting to enqueue an item to this
   channel, it wakes up that sender and get the item. Note, it still maintains
   the order. Check the details of function `recv`.
3. After step 2, there are only two cases: (1) channel is closed and its buffer
   has data. (2) the channel is open and no sender is waiting. So the rest of
   this function will try to get item from channel buffer or block the channel
   to wait for new sender.

In summary, `ok` will be false only when the channel is closed and empty, and
in this case, `x` is a zero value.

### For Range Channel

```
for c := range ch {}
```

How is this `for range` code compiled? The parser code is
[here](https://github.com/golang/go/blob/7ba074fe43a3c1e9a35cd579520d7184d3a20d36/src/cmd/compile/internal/walk/range.go#L265).
I pasted this code block to chatgpt and got a very satisfactory explanation.
Let's paste the code below since it is short. I removed all original comments.

```golang
	case k == types.TCHAN:
		ha := a

    // create temp variable hv1 with type of the channel element.
		hv1 := typecheck.TempAt(base.Pos, ir.CurFunc, t.Elem())
		hv1.SetTypecheck(1)
		if t.Elem().HasPointers() {
			init = append(init, ir.NewAssignStmt(base.Pos, hv1, nil))
		}
    // create a bool temp variable hb.
		hb := typecheck.TempAt(base.Pos, ir.CurFunc, types.Types[types.TBOOL])

    // set the condition of this for loop to `hb != false`
		nfor.Cond = ir.NewBinaryExpr(base.Pos, ir.ONE, hb, ir.NewBool(base.Pos, false))
		lhs := []ir.Node{hv1, hb}

    // rhs is expression `<-ch`
		rhs := []ir.Node{ir.NewUnaryExpr(base.Pos, ir.ORECV, ha)}

    // OAS2RECV:
    // - O: operator
    // - AS2: Assignment with 2 results
    // - RECV: Receive meaning channel receive
    //
    // hv1, hb = <-ch
		a := ir.NewAssignListStmt(base.Pos, ir.OAS2RECV, lhs, rhs)
		a.SetTypecheck(1)

    // Set above expression as the initializer of this for loop.
		nfor.Cond = ir.InitExpr([]ir.Node{a}, nfor.Cond)
		if v1 == nil {
			body = nil
		} else {
			body = []ir.Node{rangeAssign(nrange, hv1)}
		}

    // set hv1 = nil to make it GC-ed as early as possible.
		body = append(body, ir.NewAssignStmt(base.Pos, hv1, nil))
```

Basically, the parser transforms `for c := range ch {}` to a regular loop

```
var h1v ...;
var hb bool;
for h1v, hb = <-ch; hb != false; {
  ...
  h1v = null
}
```

As stated in the Receive Operator section, this loop only ends when the channel
is closed and empty.
