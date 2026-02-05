---
layout: post
title: 'Refresh your C++ knowledge: pointer-to-member type'
date: 2026-02-05 00:10 -0800
categories: [programming-language, cpp]
tags: [cpp, std, deque]
---

Even I have been writing C++ for over 10 years, I still miss a lot of basic
syntax of C++. Today, I am about to cover pointer-to-member operator. See the
standard [expr.mptr.oper](https://eel.is/c++draft/expr.mptr.oper). It is hard
to see what is going on with `.*` and `->*` without any examples. Fortunately,
cppconference has quite a lot of
[examples](https://en.cppreference.com/w/cpp/language/pointer.html).

```cpp
struct C { int m; };

int main() {
    int C::* p = &C::m;          // pointer to data member m of class C
    C c = {7};
    std::cout << c.*p << '\n';   // prints 7
    C* cp = &c;
    cp->m = 10;
    std::cout << cp->*p << '\n'; // prints 10
}
```

See the `c.*p` and `cp->*p` above. Tbh, this is my first time seeing a
pointer-to-member type `int C::*`. If you take a few seconds to think about
this syntax, then it makes total sense, right? A regular pointer is declared as
`int *`, now with class members, we prefix `*` with `C::`.

Similar syntax exists for member function as well. Check the examples in the
link above.

How could this be useful? Claude tells me it is useful for generic programming.

```cpp
struct Person {
    std::string name;
    int age;
};

void print(const Person& p, std::string Person::* field) {
    std::cout << p.*field << '\n';
}

Person person{"Alice", 30};
print(person, &Person::name);  // prints "Alice"
```

Tbh, I do not think anyone will write code this way.

### Member function has no implicit convention

The most common implicit convention in cpp is array-to-pointer and
function-to-pointer convention. LLVM semantic analysis has dedicated function
for this step
[DefaultFunctionArrayConversion](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/clang/lib/Sema/SemaExpr.cpp#L517).

But if you read the footnote of
[standard](https://eel.is/c++draft/conv#footnote-41), it says

> This conversion never applies to non-static member functions because an
> lvalue that refers to a non-static member function cannot be obtained.

That is why in the above example code, you need to write `&C::m`, but not
`C::m`. This rule sometimes creates unnecessary confusions. For example,

```cpp
void free_func(int x) { }

auto f1 = std::bind(free_func, 1);     // OK - implicit conversion
auto f2 = std::bind(&free_func, 1);    // Also OK - explicit address


struct Obj {
    void member_func(int x) { }
};

Obj obj;
auto mf1 = std::bind(&Obj::member_func, &obj, 1);  // Must use &
```

Who to blame? Not C++. It is C's fault. C++ has no choice but to make it
compatible. This
[stackoverflow post](https://stackoverflow.com/questions/72166787/why-there-is-no-implicit-conversions-to-pointers-of-member-functions?utm_source=chatgpt.com)
has a good point why this implicit convention is bad. Without decaying member
function to a lvalue, code like `if (MyClass::MyFunc)` won't compile.
