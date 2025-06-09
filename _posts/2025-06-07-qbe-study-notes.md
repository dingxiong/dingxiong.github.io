---
layout: post
title: QBE study notes
date: 2025-06-07 11:33 -0700
categories: [compiler, qbe]
tags: [compiler, qbe]
math: true
---

## SSA

"A Simple, Fast Dominance Algorithm" by Keith D. Cooper is the only paper
needed to understand SSA.

## Liveness analysis

Most my knowledge about liveness analysis comes from references [1-3]. Please
refer to these books or papers for the importance of liveness analysis. I do
not want to iterate here.

So what does it mean by being alive for a variable? Intuitively, a variable is
live if it holds a value that will/might be used in the future. A more formal
definition is

> For a CFG node q, representing an instruction or a basic block, a variable v
> is live-in at q if there is a path, not containing the definition of v, from
> q to a node where v is used. It is live-out at q if it is live-in at some
> successor of q.

There are two cases that a variable can be used.

1. It can either be used in the current block.
2. Or, it is used in one of the successor block or their corresponding
   successors if it is not redefined along the path.

Therefore, if we define `UpwardExposed(B)` as variables used in block B before
any assignment to them in B, and `Defs(B)` as all variables defined in B, then
we have formula for livein and liveout sets.

$$ LiveIn(B) = UpwardExposed(B) \cup (LiveOut(B) - Defs(B))) $$

$$ LiveOut(B) = \bigcup\_{S \in succs(B)} \{ LiveIn(B) \} $$

Minus sign above means set difference.

In SSA form, a block has zero or more phi functions at block entry and it
complicates analysis. If you treat it naively, then you will get very confused.
See this
[Reddit thread](https://www.reddit.com/r/Compilers/comments/9qt31m/comment/e8clkgi/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button).
Given blocks B1, B2, and B3 and `b3 = phi(b1, b2)` at the beginning of `B3`,
naively, you think `b1` and `b2` are used without definition, so both should
belong to `LiveIn(B3)`, and then `LiveOut(B1)` and `LiveOut(B2)` should
contains `b1` and `b2` as well. This is very counter-intuitive. A more
intuitive result should be $$ b1 \in LiveOut(B1) $$ and $$ b2 \in LiveOut(B2)

$$
respectively. There is no single valid way to deal with phi functions in
static analysis. Reference [3] section 3.3 confesses that different people use
slightly different semantics to make their algorithms work. We usually follow
[3]'s convention:

$$ LiveIn(B) = PhiDefs(B) \cup UpwardExposed(B) \cup (LiveOut(B) - Defs(B))) $$

$$ LiveOut(B) = \left( \bigcup_{S \in succs(B)} \{ LiveIn(B) - PhiDefs(S) \}
\right) \bigcup PhiUses(B)
$$

There two definitions are

- `PhiDefs(B)`: vars defined by phi nodes in B.
- `PhiUses(B)`: Vars used in phi nodes in successor blocks of B.

Please read the definition of `PhiUses(B)` carefully.

It is the phi variables in the successor blocks of B. It is not easy to
appreciate the reasoning behind the new livein and liveout formula. I gain some
intuitive from the above Reddit thread

> Phi arguments are 'used' at the end of the corresponding basic block, not at
> the beginning of the block that the phi is in.

and from [3]

> This corresponds to placing a copy of ai to a0 on each edge from Bi to B0.

In QBE, the corresponding code is inside file `live.c`

## Sparse Conditional Constant Propagation (SCCP)

## References

[1] Book "Engineering a compiler" by Keith D. Cooper. Especially chapter 8.6.1.

[2] Wikipedia
[Live variable analysis](https://en.wikipedia.org/wiki/Live-variable_analysis).

[3] Paper "Computing Liveness Sets for SSA-Form Programs" by Benoit Boissinot,
and etc.
