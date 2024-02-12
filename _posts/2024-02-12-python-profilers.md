---
layout: post
title: Python profilers
date: 2024-02-12 11:58 -0800
categories: [programming-language, python]
tags: [python, profiler]
---

CPython's thread state has a profiler callback hook
[c_profilefunc](https://github.com/python/cpython/blob/f8dc6186d1857a19edd182277a9d78e6d6cc3787/Include/cpython/pystate.h#L112).
If this callback is set, then it will be called when a function enters and
exits. See relevant code
[here](https://github.com/python/cpython/blob/955ba2839bc5875424ae745bfab53e880a9ace49/Python/ceval.c#L7249).
Node this callback is only invoked for `PyTrace_CALL`, `PyTrace_RETURN`,
`PyTrace_C_CALL`, `PyTrace_C_EXCEPTION` and `PyTrace_C_RETURN`. CPython also
has a tracer callback hook `c_tracefuncb`. This one has more overhead because
it is invoked at `PyTrace_LINE` as well.

sys module function
[sys.setprofile](https://docs.python.org/3/library/sys.html#sys.setprofile) set
the profiler for current thread. At this moment, `cProfile` is the builtin
deterministic profiler. Check relevant code
[here](https://github.com/python/cpython/blob/6b01fc7045dfeb27d146983e62759ce81ddf9e30/Modules/_lsprof.c#L406).
The official documentation claims that `cProfile` should have low overhead, but
according to
[pyinstrument](https://pyinstrument.readthedocs.io/en/latest/how-it-works.html)'s
benchmark, `cProfile`'s overhead can be as high as 84%.
