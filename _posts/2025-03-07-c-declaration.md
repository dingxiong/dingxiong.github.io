---
layout: post
title: C++ declaration
date: 2025-03-07 18:47 -0800
categories: [programming-language, cpp]
tags: [cpp]
---

## Structured Binding

Definition
https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/clang/include/clang/Sema/DeclSpec.h#L1792

https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/clang/include/clang/AST/DeclCXX.h#L4161

Parser
https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/clang/lib/Parse/ParseDecl.cpp#L7306

```
#include <iostream>
#include <type_traits>

using namespace std;


int main() {
  int a[] = {2, 3};
  auto& [x1, x2] = a;
  cout << "Is it int&: " << is_same_v<decltype(x1), int&> << endl;
  cout << "Is it int: " << is_same_v<decltype(x1), int> << endl;
  x1 = 4;
  cout << x1 << endl;
  return 0;
}

```

cppinsight version:

```
#include <iostream>
#include <type_traits>

using namespace std;

int main()
{
  int a[2] = {2, 3};
  int (&__a9)[2] = a;
  int & x1 = __a9[0];
  int & x2 = __a9[1];
  std::operator<<(std::cout, "Is it int&: ").operator<<(std::is_same_v<int, int &>).operator<<(std::endl);
  std::operator<<(std::cout, "Is it int: ").operator<<(std::is_same_v<int, int>).operator<<(std::endl);
  x1 = 4;
  std::cout.operator<<(x1).operator<<(std::endl);
  return 0;
}
```
