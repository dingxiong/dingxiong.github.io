---
title: Compiler Parser
date: 2024-01-12 14:07 -0800
categories: [compiler, parser]
tags: [compiler, parser]
---

## Lexer

### Maximal Munch principle

This principle is also called the longest token matching rule. Basically, when
there is ambiguity, the lexer always chooses the longest possible valid tokens.
Most programming languages comply with this rule. Let's take `C` as an example.
[C99 Standard (ISO/IEC 9899:1999), Section 6.4](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)
says

> If the input stream has been parsed into preprocessing tokens up to a given
> character, the next preprocessing token is the longest sequence of characters
> that could constitute a preprocessing token.

Two examples follow the above quoted sentences.

> The program fragment x+++++y is parsed as x ++ ++ +y, which violates a
> constraint on increment operators, even though the parse x +++++ y might
> yield a correct expression.

This principle is not perfect. The most famous example is operator `>>` in C++.
A template expression `A<B<C>>` fails to parse before C++11 as `>>` is taken as
a stream operator. The language standard needs to make special notes about it.

## Operator precedence parser

Parsing expressions are the most fun part of a parser. Checkout
[wiki](https://en.wikipedia.org/wiki/Operator-precedence_parser). Expression
produces value but statement does not. A good reference is from the author of
[Rust-analyze](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html).

TODO:

1. write a Pratt parser.
