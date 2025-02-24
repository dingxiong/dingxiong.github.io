---
title: C++ template
date: 2023-12-22 15:08 -0800
categories: [programming-language, cpp]
tags: [cpp, template]
---

## Template Argument Deduction For Function Calls

The ultimate source of truth is
[temp.deduct.call](https://eel.is/c++draft/temp.deduct.call). Let's take a few
examples.

### Example 1: cv-qualifier and reference type

```cpp
template <typename T> void foo(const T& x);
int x = 42;
const int y = 42;
foo(x); // A = int. P = const int&
foo(y); // A = const int. P = const int&
```

Both `int x` and `const int y` are deduced as `int`. This is because p2.3:

> If A is a cv-qualified type, the top-level cv-qualifiers of A's type are
> ignored for type deduction.

So in both cases, `A` is adjusted to `int`. The corresponding LLVM
implementation is
[here](https://github.com/llvm/llvm-project/blob/abcb66d18e3898ee42d3d313b46e18b97639a3cc/clang/lib/Sema/SemaTemplateDeduction.cpp#L4395).
Also, p3 says

> If P is a cv-qualified type, the top-level cv-qualifiers of P's type are
> ignored for type deduction. If P is a reference type, the type referred to by
> P is used for type deduction.

Here `P` is `const T&`, i.e., a reference to `const T`. Note, `const` is not at
top-level. So `P` is adjusted to `const T`. The corresponding LLVM
implementation is
[here](https://github.com/llvm/llvm-project/blob/abcb66d18e3898ee42d3d313b46e18b97639a3cc/clang/lib/Sema/SemaTemplateDeduction.cpp#L4346).
Now, the task is to solve equation `int = const T`. It seems impossible because
the extra `const` modifier, but then how compiler deduces `T = int`? This is
because p4.1

> If the original P is a reference type, the deduced A (i.e., the type referred
> to by the reference) can be more cv-qualified than the transformed A.

### Example 2: forward reference

The examples given in p3 are clear, and no need to repeat here. First, it gives
a precise definition of forward reference.

> A forwarding reference is an rvalue reference to a cv-unqualified template
> parameter that does not represent a template parameter of a class template.

For a template argument to be qualified as a forward reference, it must
satisfies 3 criteria. First, it is a rvalue reference `T&&`. Second, it should
not have cv-qualifier. `const T&&` is not a forward reference. Third, the
template parameter should not belong to the class template. So basically, a
function using forward reference can only take the form
`template <class T> foo(T&& t)` or
`template <typename... Args> foo(Args&&... args)`.

Also, p3 states the special treatment of forward reference in type deduction.

> If P is a forwarding reference and the argument is an lvalue, the type
> “lvalue reference to A” is used in place of A for type deduction.

This means that for `template <class T> foo(T&& t); int x = 5; foo(x);`, we
solve `T&& = int&`, so we get `T = int&`. You can verify that with this special
rule, no matter what the input argument is, the resulting type for `T` is
either lvalue reference or rvalue reference. That is why it is initially called
`universal referencee`. Later on, the committee standardized the name to
`forward reference` to emphasize its intentional usage: forward reference is
used to forward the argument to the downstream callees.It is not supposed to be
used directly in the current function. You can see more rigid reasoning in
[N4164](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4164.pdf).

## Perfect Forwarding

Before understanding why we need perfect forwarding, let's take a look at below
code.

```cpp
int&& r = 10;  // r is an rvalue reference to the temporary value 10
r = 20;         // r is now an lvalue, since we can assign to it
```

We definitely know that `r` is a rvalue reference. But rvalue references
themselves are lvalues. `r` has a name, shows up on the left side and persists
in memory, so it is lvalue. To extend this fact, given a function
`void foo(int&& x);`, `x` has lvalue semantics inside the function body no
matter what the input argument is. This is the exact problem for below code

```cpp
#include <iostream>

void bar(int& x) { std::cout << "Lvalue reference\n"; }
void bar(int&& x) { std::cout << "Rvalue reference\n"; }
template <typename T> void foo(T&& arg) { bar(arg); }

int main() {
    int x = 42;
    foo(x);   // Expected: Lvalue reference -> Works fine
    foo(42);  // Expected: Rvalue reference -> WRONG! Calls Lvalue version
}
```

We need to find a way to perfectly pass forward reference arguments to the
callee. Perfect forwarding allows a template function that accepts a set of
arguments to forward these arguments to another function whilst retaining the
lvalue or rvalue nature of the original function arguments. It goes like this

```cpp
template<class T>
void foo(T&& a) {
  bar(std::forward<T>(a));
}
```

Implementation of `std::forward` is simple. It is just a
[static_cast](https://github.com/llvm/llvm-project/blob/abcb66d18e3898ee42d3d313b46e18b97639a3cc/libcxx/include/__utility/forward.h).

## Tricks

Use `class... Args` to mimic Python `*args` and `**kwargs`. See
[example](https://github.com/mysql/mysql-server/blob/066cdfcf6a4523deda65a8315e12e8dfd7d49c9d/sql/iterators/timing_iterator.h#L221).
