---
layout: post
title: Golang -- Gloabl Variables
date: 2025-01-12 12:54 -0800
categories: [programming-language, golang]
tags: [golang]
---

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
