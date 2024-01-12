---
title: LLVM data structures
date: 2024-01-12 14:03 -0800
categories: [compiler, llvm]
tags: [compiler, llvm]
---

## SmallVector

[SmallVector](https://github.com/llvm/llvm-project/blob/684f3c968d6bbf124014128b9f5e4f03a50f28c5/llvm/include/llvm/ADT/SmallVector.h#L1199)
has a smart design. I was amazed the first time reading the implementation. It
utilizes the memory layout of derived classes to achieve the separation of
logic and data. `SmallVector` inherits from two classes: `SmallVectorImpl` and
`SmallVectorStorage`. The former containers vector implementation such as
`push_back`, `size()` etc. The latter is a stack storage that can hold a few
elements of this vector. When vector size grows beyond this stack storage
limit, it allocates memory from heap and the stack is not used. That is why it
is called `SmallVector` but not restricted to be small.

So how it achieve the separation? How could the implementation logic and
storage be separated into two classes? The answer is the memory layout. When a
class inherits two classes, the memory layouts of the two parent classes are
consecutive. So using the `offsetof` trick, `SmallVectorImpl` can refer to
memory that immediately after it.

```
 ------------------------
|                        |
|   SmallVectorImpl      |
|                        |
 ------------------------
|                        |
|   SmallVectorStorage   |
|                        |
 ------------------------

```
