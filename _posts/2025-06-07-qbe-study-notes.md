---
layout: post
title: QBE study notes
date: 2025-06-07 11:33 -0700
categories: [compiler, qbe]
tags: [compiler, qbe]
math: true
---

## Liveness analysis

Most my knowledge about liveness analysis comes from book "Engineering a
compiler" chapter 8.6.1 and Wikipedia
[Live variable analysis](https://en.wikipedia.org/wiki/Live-variable_analysis).
Why we need liveness analysis? First, it helps with register allocation.
Registers are limited resources. Variables with no-overlapping liveness range
can reuse registers. Second, liveness analysis is used to determine
uninitialized variables and dead code. It is good for semantic analysis and
optimization.

So what does it mean by being alive for a variable? A variable is live if it
holds a value that will/might be used in the future. A more formal definition
is

> A variable v is live at point p if and only if there exists a path in the cfg
> from p to a use of v along which v is not redefined

We need to two concepts that help us understand the iterative algorithm for
computing liveness.

> Livein definition: a variable (temp) a is live-in at node n if it is used at
> n before any assignment in the same basic block, or if there is a path from n
> to a node that uses a that does not contain a definition of (assignment to)
> a.

> Liveout definition: a variable a is live-out at node n if it is live-in at
> one of the successors of n.

The sets in[n] and out[n] satisfy the equations

$$ in[n] = use[n] \cup (out[n] - def[n]) $$

$$ out[n] = \bigcup \{ in[s] | s \in succ[n] \} $$

In SSA form, a block has zero or more phi functions at block entry. For an
example phi function `x3 = phi(x1, x2)`. `x1` and `x2` are used before
definition, therefore, they belong to the livein set of current block, and
liveout set of the predecessor node. `x3` is defined at the start of the block,
so it is definitely not in livein set of current block

Man. I am lost in these formulas. I will come back.

In QBE, the corresponding code is inside file `live.c`

## SSA
