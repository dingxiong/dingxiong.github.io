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

## Basic

Character literals starts with `#\`. For example, `#\Backspace` denotes a
backspace, and `#\'` denotes a single quote.

Reader Macros

[Common Lips style guide](https://lisp-lang.org/style-guide/):

- Constants should be surrounded with plus signs
