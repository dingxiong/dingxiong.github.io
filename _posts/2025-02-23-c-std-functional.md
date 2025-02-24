---
layout: post
title: C++ std -- functional
date: 2025-02-23 20:48 -0800
categories: [programming-language, cpp]
tags: [cpp, std, functional]
---

Various components inside header `<functional>`.

## std::less and std::greater

These are two structs with
[trivial implementation](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/__functional_base#L53)
of `operator()`. It relies on the value type to implement `operator<` or be a
primitive type such as `int`.

Let's how std containers implement `operator<`. Before that, let's quickly take
a look at
[std::lexicographical_compare](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/algorithm#L5548).
The implementation basically says that it compares two containers from left to
right until it finds `a[i] < b[i]` or `a` is shorter than `b`.

First, let's check `std::array`. The definition is
[here](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/array#L387).
It basically calls `std::lexicographical_compare`. However, we need to be
cautious that this comparator is defined only for the same array type. The
signature is
`operator<(const array<_Tp, _Size>& __x, const array<_Tp, _Size>& __y)` not
`operator<(const array<_Tp, _Size>& __x, const array<_Tp2, _Size2>& __y)`. Both
the type and size should be the same, otherwise, you will see a compilation
error.

Second, let's check `std::vector`. The definition is
[here](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/vector#L3352).
Similar to `std::array`. The two vectors should have the same type. However, it
allows to have different size. Namely
`std::vector<int>{1, 2} < std::vector<int>{1, 2, 1}`.

Third, `std::pair`. OK. it has its own
[implementation](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/utility#L591-L592),
but still trivial enough.

Forth, `std::tuple`. It uses a SFINAE trick
[implementation](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/tuple#L1191).
