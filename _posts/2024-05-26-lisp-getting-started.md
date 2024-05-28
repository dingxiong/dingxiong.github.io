---
layout: post
title: Lisp -- Getting Started
date: 2024-05-26 09:54 -0700
categories: [programming-language, lisp]
tags: [lisp]
---

I took a deep breath when starting this blog post. Lisp is such an arcane
programming language. I am not sure I will benefit from it at all. However,
since `pgloader` is written in CL, I think I should have a try.

Most of the following is about Common Lisp.

SBCL

ASDF

Quicklisp

## Reader Macros

Reader Macros is kind of similar to macros in C. This
[blog post](https://lisper.in/reader-macros) shows how to use it to modify the
CL parser to parse json. Often in a CL project, you will see code such as
`#+sbcl` or `#+ccl`, which is used to conditionally include/exclude code based
on whether sbcl or ccl is used. The counterparty in C is `#ifdef xxx` or
`if constexpr` in C++.

As a beginner, I was quite confused by the usage of special characters such as
`#\`, `#:`, `+name+`, `*name*` etc. Most of them are reader macros! Common Lips
calls them "dispatching macro character" more specifically. In Common Lisp, a
dispatching macro character is a type of macro character that dispatches to
another function based on the character that follows it.

Sharp (`#`) is the mostly frequently used dispatching macro. CLHS has a chapter
for it: [link](http://clhs.lisp.se/Body/02_dh.htm). Some common ones are list
below.

1. `#\`: character literal. For example, `#\Backspace` denotes a backspace, and
   `#\'` denotes a single quote.
2. `#(`: simple vector. For example: `#(a b c )`
3. `#:`: uninterned symbol.
4. `#+`: read-time conditionalization facility.

Some of these special character are not reader macros, but pure styling
convention. See [Common Lips style guide](https://lisp-lang.org/style-guide/):

- Constants should be surrounded with plus signs
- Special variables (Mutable globals) should be surrounded by asterisks.
