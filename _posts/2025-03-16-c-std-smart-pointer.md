---
layout: post
title: C++ std -- smart pointer
date: 2025-03-16 14:35 -0700
categories: [programming-language, cpp]
tags: [cpp]
---

## shared_ptr and weak_ptr

shared_ptr uses reference counting to realize automatic memory management.
shared_ptr is very light-weighted. It only has
[two member variables](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L323):
a pointer to the managed object and a pointer to the control block.

There are mainly two ways to create a shared*ptr:
`shared_ptr<T>(new T(Args...))` and `make_shared<T>(Args...)`. For the first
method, the object is created by `new`, and then a control block  
[\_\_shared_ptr_pointer ](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L96).
The control block contains a compressed triple: `(ptr*, **deleter,
**alloc*)`and two counters `shared_weak_owners*`and`shared*owners*` inherited
from the base class. Because the object and control block are constructed one
after another. So they most likely do not sit next to each other in the memory
layout.

On the other hand, in the second method, the object is constructed together
with the control block. The control block has name
[shared_ptr_emplace](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L144).
The control block and object are stored in a consecutive byte array. See
[code](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L202).
So this method is more cache friendly. Also, the control block does not need to
maintain a pointer to the object. It save some space.

### How does reference counting work?

No matter how we create the share_ptr, the control block is subclass of
`__shared_weak_count` which is a subclass of `_shared_count`. There are two
virtual functions inside: `__on_zero_shared_weak` and `__on_zero_shared`. They
controls the behavior of time the shared counter and weak count become zero.
When shared counter goes to zero, the object is destructed and when weak
counter goes to zero, the control block is destructed. This does not sound
correct because if we never created a weak_ptr from a shared_ptr, then when the
control block is freed? This
[code](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_count.h#L120)
answers this question. Basically, we the shared counter goes to -1, we also
check if weak counter is zero or not. If it is, it means no weak_ptr and the
control block is destructed as well.

Initially, I was confused by the difference of number `-1` and `0`. After
adding some print logs and rebuild libcxx helps me understand what is going on.
Let's see below example. `sp1` means a shared_ptr is created, and `~wp1` means
a weak_ptr is destroyed.

```
timeline:      start -> sp1 -> wp1 -> wp2 -> ~sp1    ->    ~wp1   -> ~wp2
shared_count:  0        0      0      0       -1            -1       -1
weak_count:    0        0      1      2        2             1        0
event:                                       object                  control block
                                             destroy                  destroy
```

The trick is that when a shared_ptr is first created, shared counter is zero.
only when a second share_ptr is created, the counter increases to 1. On the
other hand, the first time we create a weak_ptr from a shared_ptr, the weak
counter is increased to 1.

Now we understand the design of shared_ptr. It is a thin structure that
contains two pointers. All related shared_ptr and weak_ptr point to the same
control block and the same object.

With the knowledge so far, let's take a look at two functions in a weak_ptr.
`expired()` returns whether the managed object has been deleted or not. It is
equivalent to
[use_count == 0](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L1254).
Wait? In the above diagram, we see that the condition should be
`shared_count == -1`. Something is wrong? No. It turns out
[use_count = shared_count + 1](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_count.h#L98).

Another function is `weak_ptr::lock`. It creates a new shared_ptr if the
managed object is not deleted yet. Otherwise, it returns an empty shared_ptr.
The lock implementation is
[here](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/src/memory.cpp#L88).
Man, it is a spin lock! If thousands of threads try to call `weak_ptr::lock` at
the same time, then it is totally possible that this while loop will stuck for
a while.

### std::enable_shared_from_this

I still remember the stress that I saw this class used a in PR when I was at
Citadel. That is the first time I saw it and I do not have time to figure it
out. Now I am at a different company and have the free time to figure it out.

First, why we need this? Suppose you have a class T, and inside it you defined
a function `foo`, you want `foo` to return a `shared_ptr<T>` of this class
itself. What you will do? `return shared_ptr<T>(this)`? No way, this leads to
double free if this class is used this way:
`p1 = make_shared<T>(...); p2 = p1->foo();`. The two shared pointers do not
share the control block!

The correct way is

```cpp
class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    std::shared_ptr<MyClass> foo() { return shared_from_this(); }
};
```

class `enable_shared_from_this` has only one member variable
[weak_ptr weak_this](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L1436).
All `shared_ptr` constructors must do one thing: call
[enable_weak_this](https://github.com/llvm/llvm-project/blob/f5f5286da3a64608b5874d70b32f955267039e1c/libcxx/include/__memory/shared_ptr.h#L695).
Note this function is templated with two variants. If the current class is a
subclass of `enable_shared_from_this`, then it initializes variable
`weak_this`. Please read carefully, the implementation of this function uses a
`share_ptr` constructor that shares the control block with itself, so we do not
have double free problem. If the current class is not a subclass of
`enable_shared_from_this`, then this function is a no-op. So the net effect is
that your class automatically gets a weak pointer to itself if it inherits
`enable_shared_from_this`.
