---
layout: post
title: C++ std -- unordered set/map
date: 2025-02-23 20:43 -0800
categories: [programming-language, cpp]
tags: [cpp, std, unordered_set, unordered_map]
---

## `std::unordered_set`

Both `unordered_set` and `unordered_map` use an internal
[\_\_hash_table](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/__hash_table#L751)
object, so it suffices to only talk about how hash table works in C++.

The basic structure looks like diagram below. It contains two parts: a
single-linked list of nodes and an array of bucket head.

```
node1 --> node2 --> node3 --> ... --> node_n --> nullptr
  |                  |
  |                  |
[bucket1,           bucket2, ....           ]
```

The
[\_\_bucket_list](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/__hash_table#L807)
is simply an raw array. Each array element points to a node in the listed list.
So the nodes between `bucket_i` and `bucket_{i+1}` belong to `bucket_i`.

Each node in the linked list contains a value. For `unordered_set`, its type is
`class Key`. For `unordered_map`, it is `pair<class Key, class Value>`. It is
unlike Java which uses `HashMap` to simulate `HashSet`, which needs an
unnecessary value field. C++ is more space efficient.

There are two hash concepts in `__hash_table`. One is node hash. The node value
type should define a function `__hash()` which is used to get the node hash.
Another is bucket hash `std::__constrain_hash(node_hash, bucket_count)`. It
takes node's hash and total number of buckets as input and return the bucket id
this node belows to. In the simplest case, it is `node_hash % bucket_count`.

### How does `::erase` work?

Function `erase` calls
[remove](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/__hash_table#L2321).
Basically,

1. find the previous node of the node to be erased in the linked list.
2. Deal with special cases: prev node is in the previous bucket, or the next
   node is in the next bucket. In these edge cases, after deleting the current
   node, the bucket becomes empty, so we need to delete the bucket as well.
3. Delete the node: `prev->next = curr->next; curr->next = null`.

Note, the function returns a `__node_holder`.

```
    return __node_holder(__cn->__upcast(), _Dp(__node_alloc(), true));
```

So, the deleted node will not be destructed immediately. It is up to the
calling site how to deal with this node. Usually, the calling site does
nothing, and `_Dp(__node_alloc())` destructs this node immediately once it goes
out of scope.

## `std::unordered_map`

### `operator[]`

The
[specification](https://en.cppreference.com/w/cpp/container/unordered_map/operator_at)
says the value_type must be `DefaultConstructible`. Let's say what happens if
it is not. For a simple program as below

```cpp
struct A {
  int a;
  A(int a_): a(a_) {}
};

int main() {
  unordered_map<int, A> u;
  auto x = u[0];
  cout << "value:" << x.a << std::endl;
}
```

The compilation errors is

```
error: no matching constructor for initialization of 'A'
      second(_VSTD::forward<_Args2>(_VSTD::get<_I2>(__second_args))...)
```

Let's take a deeper look. The definition for `operator[]` is
[here](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/unordered_map#L1662-L1663).
If the key is not found in the table, then it calls `__construct_node_with_key`
to construct a k-v pair to insert to `__table`. The construction process uses
`allocator_traits::construct` without constructor arguments. Namely, it calls
the default constructor of the value type.

### `iterator`

Sometimes, I mixed different iterators. Some iterators can do `it+2`, some can
only do `it++`. So how does the iterator of `unordered_map` work?

According to
[cppreference](https://en.cppreference.com/w/cpp/container/unordered_map), the
iterator of `unordered_map` is a `LegacyForwardIterator`. It has only two
operations: `operator++` and `operator++(int)`. See
[source code](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/unordered_map#L993).

However, the `<iterator>` header defines a function
`std::advance(InputIt& it, Distance n)`. How does this work? The source code is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/__iterator/advance.h#L44).
Basically, depending on the iterator type, it can choose to directly add `n` to
the base iterator, or use a loop to advance the iterator.

Btw, what type of iterator do other containers have?

- LegacyForwardIterator: unordered_set, unordered_map,
- LegacyBidirectionalIterator: set, map
- LegacyRandomAccessIterator: vector

For `set`, `map`. It uses a tree underneath, the iterator definition is
[here](https://github.com/llvm/llvm-project/blob/ab6d5fa3d0643e68d6ec40d9190f20fb14190ed1/libcxx/include/__tree#L877).
It defines both `++` and `--` operators.
