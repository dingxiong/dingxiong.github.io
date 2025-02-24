---
layout: post
title: C++ std -- variant
date: 2025-02-23 20:47 -0800
categories: [programming-language, cpp]
tags: [cpp, std, variant]
---

One lesson I learnt from reading `std::variant` implementation is that `union`
in C++ can have complicated structure. Before, I thought we should only use
union in this way:

```cpp
union U {
    int a;
    char b;
    ...
};
```

According to [cpp standard](https://en.cppreference.com/w/cpp/language/union),
the body can be

> list of access specifiers, member object and member function declarations and
> definitions.

Look at the
[variant base class](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/variant#L758),
you see it defines constructors, destructor, and the data members are private.

To make it work at compile time, the implementation definitely need to use a
lot of meta programming. Damn it! I hope one day my brain can natively parse
these recursions.

## How does `std::get<Tp>(variant)` work?

`std::variant` provides two get function to obtain the underlying union value:
get-by-index and get-by-type. Get-by-index is natively
[supported](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/variant#L1512)
because underneath, variant is just a union which contains
[a value and index](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/variant#L737-L738).
Then how is get-by-type supported? Naturally, we need to find the index of this
type in the variant definition. The implementation is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/variant#L1580)
and
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/tuple#L1448).
It is quite interesting because variant reuses something from tuple.

## How does `std::visit` work?

Visit pattern is verbose. In simple cases, I would avoid it for simple variant.
For example, for `std::variant<int, string>`, I can implement the output stream
operation such as

```cpp
ostream& operator<<(ostream& os, const std::variant<int, string>& v) {
    if (v.index() == 0) os << std::get<0>(v);
    else os << std::get<1>(v);
    return os;
}
```

Note, `std::visit` function not only accept one variant, but also multiple
variants. If more than one variant is supplied, the visitor function is
supposed to work on the product space. See below example.

```cpp
using U = std::variant<char, bool>;
using V = std::variant<int, string>;

struct Visitor {
  std::string operator()(char a, int b) { return to_string(a) + "-" + to_string(b); }
  std::string operator()(char a, const string& b) { return to_string(a) + "-" + b; }
  std::string operator()(bool a, int b) { return to_string(a) + "-" + to_string(b); }
  std::string operator()(bool a, const string& b) { return to_string(a) + "-" + b; }
};

int main(int argc, char** argv) {
  U a = true;
  V b = 2;
  cout << std::visit(Visitor{}, a, b) << endl;
}
```

How is this product space visitor implemented? I would guess the meta
programming skills needed are crazy. Take a look at
[this code](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/variant#L533)
if you are interested. Basically, it constructed an index matrix.
