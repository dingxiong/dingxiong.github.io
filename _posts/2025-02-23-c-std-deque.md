---
layout: post
title: C++ std -- deque
date: 2025-02-23 20:49 -0800
categories: [programming-language, cpp]
tags: [cpp, std, deque]
---

## `pop_front`

Below code has a bug

```cpp
auto& element = que.pop_front()
```

According to
[the standard](https://en.cppreference.com/w/cpp/container/deque/pop_front),
reference to the popped element will be invalidated, so we should create a copy
instead of reference. The implementation is
[here](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/deque#L2706).
You can see that is calls `allocator_traits::destroy` to free the object.
