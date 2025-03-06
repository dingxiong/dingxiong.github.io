---
layout: post
title: Python -- weakref
date: 2025-03-06 12:15 -0800
categories: [programming-language, python]
tags: [python, weakref]
---

`weakref.ref` and `weakref.ReferenceType` are the same thing. See
[code](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Modules/_weakref.c#L154).

The `__call__` is implemented
[here](https://github.com/python/cpython/blob/878ead1ac1651965126322c1b3d124faf5484dc6/Include/cpython/weakrefobject.h#L51).
