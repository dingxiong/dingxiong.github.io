---
layout: post
title: C++ std -- vector
date: 2025-02-23 20:46 -0800
categories: [programming-language, cpp]
tags: [cpp, std, vector]
---

I got quite confused the first time I read
[vector](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/vector)
implementation. After a few minutes' struggle, I realized that there are two
implementations: the general `vector` and the specialized `vector<boo>`. There
are quite a few differences between them. For example, the `::clear()`
function. The general implementation is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/vector#L720).
The bool specialized version is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/vector#L2364).
You see the difference. The former call destructor. The latter does not.

## `std::vector<T, Allocator>::clear()`

My first question about `clear` function is whether it calls the destructor.
For the
[general implementation](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/vector#L720),
yes, it does.

Let's analyze a simple program

```cpp
struct A {
  int n;
  A(int n):n(n){}
  ~A() {cout << "destroy " << n << endl;}
};

int main() {
  vector<A> x;
  cout << "-- delimiter 0 ---" << endl;
  x.emplace_back(1);
  cout << "-- delimiter 1 ---" << endl;
  x.emplace_back(2);
  cout << "-- delimiter 2 ---" << endl;
  x.clear();
  cout << "-- delimiter 3 ---" << endl;
}
```

The output is

```
$ g++ test.cc -std=c++17 && ./a.out
-- delimiter 0 ---
-- delimiter 1 ---
destroy 1
-- delimiter 2 ---
destroy 2
destroy 1
-- delimiter 3 ---
```

You see that After calling `x.clear()`, both elements are destroyed.

Wait, why the first element is destroyed twice? This is because, when pushing
the second the element, it goes beyond its internal capacity, so it needs to
creates a new buffer and copy existing elements to the new buffer, and during
this process, the old elements will be destroyed. If you add `x.reserve(2);` at
the beginning, then you won't see it.
