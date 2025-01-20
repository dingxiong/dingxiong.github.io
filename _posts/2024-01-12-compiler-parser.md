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

### Implementation details

```
Interface Lexer {
  boolean hasNext();
  Token next();
  Token peek();
}
```

## Pratt Parser (Operator Precedence Parser)

Parsing expressions is the most fun part of a parser. There are many ways to do
it. Both BNF based parser and
[Pratt parer](https://en.wikipedia.org/wiki/Operator-precedence_parser) can
parse expressions. Then what is the benefit of the latter? It is because Pratt
parser is simpler and more performant. Checkout this
[answer on stackexchange](https://langdev.stackexchange.com/a/3275/6860).
Basically, Pratt parser replaces deep nested function calls with a single loop.
A good reference implementation is from the author of Rust-analyze,
[Alex Kladov](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html).
Another implementation is from the creator of Javascript,
[Douglas Crockford](https://www.crockford.com/javascript/tdop/tdop.html).

The two major challenges of parsing expressions are operator precedence and
association direction. Pratt parser solves them by `operator binding power`.
Each operator has a left and a right binding power: `lbp` and `rbp`. Also, each
operator can be classified into three categories: prefix, infix, and postfix.
Some operators can be both prefix and infix and others can be both postfix and
infix. Prefix operators have a `nud` method. Infix and postfix operators have a
`led` method. Douglas Crockford's pseudocode below is the essence of Pratt
parser.

```js
var expression = function (rbp) {
  var left;
  var t = token;
  advance();
  left = t.nud();
  while (rbp < token.lbp) {
    t = token;
    advance();
    left = t.led(left);
  }
  return left;
};
```

To parse an expression, we parse its first token to an expression node called
`left`. Line 5 `t.nud()` will recursively call `expression(rbp)`. If the next
token after `left` is `EOF`, then we are done, we return left. If not, then
this token must be an operator, and must be either postfix or infix operator.
The rest is is a loop that tries to find the right boundary of this left
expression. More specifically, we need to figure out whether it is
`(... left) op tok1 tok2 ...` or `(...) (left op tok1 tok2 ... tokn) ...`. If
`op.lbp` is less than the function input `rbp`, then we are sure it is the
first case: there is a boundary between `left` and `op`. In this case, we
return `left`. If not, then it means `left` and `op` should be bundled together
and we need to parse the right hand side, which is handled by line 9
`left = t.led(left)`.

Some operators are easy to understand such as `+` and `-`. Others may need some
special treatment.

- `[]` is an infix operator because it has form `a[b`. Then how is `]` parsed?
  Let's look at the details of the parsing steps of `a[b]`. Suppose the current
  token is `b`. After line 4 `advance()` is called, `token = ]`. Then the
  condition for the while loop does not hold. Here we assume `].lbp = -1` or we
  can set `].lbp = null` and change the condition to
  `token.lbp is not null && rbp < token.lbp`. So we directly return `left`
  which is just `b`. This means the right side of `[` is `b` and in the outer
  call, `left=a[b` and current token is `]`. At this moment, we just need to
  discard current token. In Alex Kladov's implementation, he emits an assert
  statement `assert_eq!(lexer.next(), Token::Op(']'));`.
- `()` can be prefix or infix. When used as grouping operator, the form is
  `(a`, so it is prefix. When used as function call, the form is `f(a`, so it
  is infix. The way `)` is handled is similar to `]`. The only purpose of `)`
  is to indicate the right boundary. It is discarded in the end.
- `.` is infix because its form is `a.b`.
- `?:` is infix because its form is `a?b`. You can think of `?:` the same as
  `[]`. The difference is that we are done after discarding `]`. But after
  discarding `:`, we need to parse another expression and it is required.
- Assignment `=` is also an infix operator.

TODO:

1. write a Pratt parser.
