---
layout: post
title: C++ std -- range
date: 2025-03-16 02:25 -0700
categories: [programming-language, cpp]
tags: [cpp]
---

Range
[libcxx implementation](https://github.com/llvm/llvm-project/tree/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__ranges).

Let's walk through a simple example.

```cpp
#include <iostream>
#include <ranges>
using namespace std;

int main() {
  vector<int> a = {1, 2, 3, 4};
  int n = 3;
  for (const auto& x : a | views::take(n)) { cout << x << ","; }
  return 0;
}
```

The first note is about the pipe operator. It acts between an vector and a
range adaptor. The definition in libcxx is
[here](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__ranges/range_adaptor.h#L73).
The left side of `|` is any type that satisfies
[range concept](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__ranges/concepts.h#L47),
namely, any type that defines the `begin` and `end` functions. Actually, my
statement is not rigorous. The range library defines 4 valid cases. See
[code](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__ranges/access.h#L65).
So it works too if we change vector to `int a[] = {1, 2, 3, 4};` The right side
is a range adaptor, which is simply a functor. `x | f` means `f(x)` and the
result is a new range.

Secondly, how does `views::take(n)` works? The definition is
[here](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__ranges/take_view.h#L347).
It uses a C++23 feature `std::bind_back` to store the parameter `n` in the
functor and then call `views::taken(a, n)`. The final result can be the
original range type, or iota range, take_view, etc. For the above case, the
result is a `take_view`. If we change `a` to be `auto a = views::iota(1, 5)`,
then the result is a `iota_view`.
