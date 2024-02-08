---
layout: post
title: compiler introduction
date: 2024-01-12 14:08 -0800
categories: [compiler]
tags: [compiler]
---

Recently, I spent some time learning compiler techniques. First, I read the
first few chapters of dragon book. Then I turned to blogs teaching me how to
build a compiler in short time.

After a short surfing, I found no blogs beat Bison's manual and the examples in
it. So I read it through carefully. Later, I found a good blog
https://thinkingeek.com/archives/

## Lexical analyzer & Parser

There are a lot of blogs online with title "writing your own parser in language
xxx". Just follow one of them to get familiar with the basic usage of `flex`
and `bison`. Why these two? There are many "better" alternatives like `ANTLR`
and those written in some popular non-C language like `goyacc`, but
`flex + bison` are classical.

[Flex's manual](https://westes.github.io/flex/manual/index.html#SEC_Contents)
is very short. You can quickly get an idea of what it is doing.
[Bison's manual](https://www.gnu.org/software/bison/manual/bison.html), on the
other hand, is long. But there is lots of meat inside. The concept section is a
refresh of what you learned in the dragon book.

Bison's manual has three examples. Just follow them to get started.

### Static single-assignment form (SSA form)

## Instruction selection

Gabriel's paper
[Survey on Instruction Selection: An Extensive and Modern Literature Review](https://arxiv.org/abs/1306.4898)
is an extensive report on the instruction selection techniques. To summarize,
macro expansion, tree covering and DAG covering are among the most popular ways
to do instruction selection. LLVM uses DAG covering and chapter 4.3.3 has a
paragraph talking about it.

On the other hand, I recently find a small compiler backend called
[QBE](https://github.com/8l/qbe/tree/master). It does instruction selection by
macro expansion. See
[arm64/emit.c](https://github.com/8l/qbe/blob/d2075d5e131e181820d8c39ad74322f970d562aa/arm64/emit.c#L43).
Each operand maps to a instruction macro string with special symbols `%0`,
`%=`, etc to be filled out later.

## References

- [A Simple, Possibly Correct LR Parser for C11](https://dl.acm.org/doi/10.1145/3064848)

# TODO:

1. Write a markdown parser.
2. Study Clang parser.
3. write a compiler
4. write a interpreter.
