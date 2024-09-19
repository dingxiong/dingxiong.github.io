---
layout: post
title: Golang -- Getting started
date: 2024-04-19 19:34 -0700
categories: [programming-language, golang]
tags: [golang]
---

Follow [this guild](https://go.dev/doc/install/source) to build golang from
source.

```
cd src && ./all.bash

# If you do not want to run tests, then
./make.bash
```

Also, remember to clear cache `go clean --cache` after rebuilding go, or use
`-a` argument when running `go build`.

## Global variables

As a newbie to Golang, I am quite impressed by the flexibility of global
variables inside Golang. Below simple C program fails to compile.

```c
#include <stdio.h>
int foo() { return 5; }
int a = foo();

int main(int argc, char** argv) {
  printf("%d\n", a);
  return 0;
}
```

The error message is

```
gv.c:7:9: error: initializer element is not constant
    7 | int a = foo();
```

However, this is a valid pattern in Golang, so how does it work? I will use go
1.19.13 to illustrate package-level variables in Golang.

```go
package main;
import "fmt"

var WhatIsThe = AnswerToLife()

func AnswerToLife() int { return 42 }

func init() { WhatIsThe = 0 }

var b = a
var a = 100

func main() {
  fmt.Println(WhatIsThe, b)
}
```

The above program compiles successfully and the output is `0 100`. A few things
to note. First, variable `WhatIsThe` are initialized twice: when it is defined
and inside the init function. The latter assignment overwrites the former's
result. Second, variable `b` is assigned the value from `a` but defined before
`a`. How is this possible?

The main logic is inside `init.go`. I added some loggings in this file to help
me understand what is going on.

```
diff --git a/src/cmd/compile/internal/pkginit/init.go b/src/cmd/compile/internal/pkginit/init.go
index 8c60e3bfd6..81b833c4d9 100644
--- a/src/cmd/compile/internal/pkginit/init.go
+++ b/src/cmd/compile/internal/pkginit/init.go
@@ -14,6 +14,7 @@ import (
        "cmd/compile/internal/types"
        "cmd/internal/obj"
        "cmd/internal/src"
+  "fmt"
 )

 // MakeInit creates a synthetic init function to handle any
@@ -139,6 +140,7 @@ func Task() *ir.Name {

        // Record user init functions.
        for _, fn := range typecheck.Target.Inits {
+    fmt.Printf("%v: %v\n", fn.Sym().Name, fn.Body)
                if fn.Sym().Name == "init" {
                        // Synthetic init function for initialization of package-scope
                        // variables. We can use staticinit to optimize away static
@@ -169,6 +171,7 @@ func Task() *ir.Name {
                fns = append(fns, fn.Nname.Linksym())
        }

+  fmt.Println("===", deps, fns)
        if len(deps) == 0 && len(fns) == 0 && types.LocalPkg.Path != "main" && types.LocalPkg.Path != "runtime" {
                return nil // nothing to initialize
        }
```

Then running `go build`, I got below logs

```
init: WhatIsThe = ~R0; a = 100; b = a
init.0: WhatIsThe = 0
=== [fmt..inittask] [main.init main.init.0]
```

At the build stage, Golang compiler wraps package-level variables inside an
`init` function. See function
[MakeInit](https://github.com/golang/go/blob/go1.19.13/src/cmd/compile/internal/pkginit/init.go#L24).
The first line of this function `nf := initOrder(typecheck.Target.Decls)`
calculates the order of package-level declarations so that `b` will be
initialized after `a` in our example. By the way, this function does not check
circular dependency, which is checked much earlier in a function with the same
name but belonging to a different package.

For the output above, you can see that Golang compiler generates two `init`
functions: `main.init` from package-level declarations and `main.init.0` from
the `init` function, and `main.init` goes before `main.init.0`. That is why
`WhatIsThe = 0` in the output.

You may notice the logging line also outputs `deps`, which is `fmt..inittask`
in our case. Why does it need this information? When there are multiple
packages and there are dependencies, the compiler needs to figure out the order
to run these package-level init tasks. I do not have time to figure out how
this part is implemented in go 1.19.13, but this
[commit](https://github.com/golang/go/commit/ce2a609909d9de3391a99a00fe140506f724f933)
adds a file `inittask.go`, which does topological sorting to figure out the
import order.

The compiler generates the `main.init` and `main.init.0` functions. That is
good, but how are they called? We can check the assembly output of the
generated binary by `go tool objdump -s '^main*' <binary>`. The output contains
three labels: `main.main`, `main.init` and `main.init.0`. The Golang runtime
ensures that the latter two run before `main.main`. (TODO: figure out how
runtime calls them in this order.)

OK, coming back to the original C example. The Golang equivalent one is

```c
#include <stdio.h>
int foo() { return 5; }
int a;

void main_init() {
  a = foo();
}

int main(int argc, char** argv) {
  main_init();
  printf("%d\n", a);
  return 0;
}
```

## Channel

Channel is usually used together with goroutines. Let's take an example from
[k8s source code](https://github.com/kubernetes/kubernetes/blob/1850794626bb995bb754d54be3328c86ee880ba5/vendor/github.com/opencontainers/runc/libcontainer/notify_linux.go#L20).
This function returns a channel for callers to consume. Inside, it spawns a
background goroutine to send messages to this channel. Note, this channel is
created outside this goroutine but closed inside it. Also, errors are all
handled inside this goroutine. It is golang's philosophy to deal with error
instead of popping up error.

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
