---
layout: post
title: C++ std -- queue
date: 2025-02-23 20:50 -0800
categories: [programming-language, cpp]
tags: [cpp, std, queue]
---

## priority queue

### Constructor

Priority queue has signature
`std::priority_queue<T,Container=vector,Compare=std::less>` , so you can see
that comparator class is provided as a template parameter. Below code does not
compile

```cpp
priority_queue<int> pq(std::greater<int>{});
```

with error message `note: candidate constructor template not viable:...`. So
how should we do it?

```cpp
auto comp = std::greater<int>();
priority_queue<int, std::vector<int>, decltype(comp)> pq(comp);
```

Ah! Nice `decltype`. Note that, we cannot omit the second template parameter
which is quite annoying.

Another note about above code is `std::greater<int>{}`. Why uses curly braces
instead of parenthesis. Checkout
[Most vexing parsing](https://en.wikipedia.org/wiki/Most_vexing_parse).
Basically, C/C++ will interpret it as

```cpp
priority_queue<int, std::vector<int>, decltype(comp)> pq(std::greater<int>);
```

i.e., a function definition `pq` with an anomalous parameter. C/C++ allow
superficial parenthesis around function parameter.
