---
layout: post
title: Rust -- Generics and Polymorphism
date: 2025-08-27 16:48 -0700
categories: [programming-language, rust]
tags: [rust]
---

Before talking about Rust, let's see what we have in C++. First, C++ has
template. Compiler generates distinct binary code for each different
instantiated template type. For example,
`template<typename T> is_positive(T x) { return x > 0; }`. We can call this
function with either floats or integers. Compiler generates two versions of
binary code. There is no redirection, up/down-casting or whatever at runtime.
On the other hand, C++ has class inheritance and virtual functions to achieve
runtime polymorphism so the same code can be used for base class and sub class
without duplication. But at runtime, we pay the cost of redirection via vtable.

This introduces the concepts of compile-time polymorphism and runtime
polymorphism. In performance critical situations, we prefer compile-time
polymorphism. How to achieve compile-time polymorphism? A naive implementation
is below.

```cpp
#include <type_traits>
#include <iostream>
using namespace std;

struct Base { virtual int foo() {return 2; }; };
struct Derived : Base { int  foo() override { return 3; } };

template <typename T>
requires is_base_of_v<Base, T>
int callFoo(T& obj) {
    return obj.foo();  // virtual dispatch (runtime polymorphism)
}

int main() {
    Base x;
    Derived y;
    cout << callFoo(x) << endl;
    cout << callFoo(y) << endl;
    return 0;
}
```

First, compiler in deed generates two functions `callFoo<Derived>(Derived&)`
and `callFoo<Base>(Base&)`. (Please turn off any optimization when checking the
generated assembly because compiler/linker can easily dedup and inline these
two functions.) However, the step `return obj.foo()` still incurs vtable
redirection at runtime because it is a virtual function.

We need CRTP (Curiously Recurring Template Pattern) to achieve true
compile-time polymorphism.

```cpp
#include <type_traits>
#include <iostream>
using namespace std;

template<typename Derived>
struct CRTPBase {
    int foo() { return static_cast<Derived*>(this)->fooImpl(); }
};

struct Base : CRTPBase<Base> {
    int fooImpl() { return 2; }
};

struct Derived : CRTPBase<Derived> {
    int fooImpl() { return 3; }
};

int main() {
    Base x;
    Derived y;
    cout << x.foo() << endl;
    cout << y.foo() << endl;
    return 0;
}
```

The critical change is that `foo()` is no longer virtual, and `static_cast`
directly downcasts the type. So there is no run-time redirection through
vtable.

So much about C++. What about Rust then? Rust has trait and trait object.

- Compile-time polymorphism in Rust = generics + monomorphization
- Runtime polymorphism in Rust = trait objects

[Monomorphization](https://en.wikipedia.org/wiki/Monomorphization) corresponds
to C++ template. Let's see a simple example.

```rust
trait Base { fn foo(&self) -> i64; }
struct Derived1;
struct Derived2;

impl Base for Derived1 { fn foo(&self) -> i64 { 2 } }
impl Base for Derived2 { fn foo(&self) -> i64 { 3 } }

fn call_foo<T: Base>(obj: &T) -> i64 { obj.foo() }

fn main() {
    let d1 = Derived1;
    println!("{}", call_foo(&d1));
    let d2 = Derived2;
    println!("{}", call_foo(&d2));
}
```

Compiling above code `rustc test.rs --emit=asm -o - | rustfilt` generates below
sections

```
_<test::Derived1 as test::Base>::foo:
        mov     w8, #2
        mov     x0, x8
        ret
        .p2align        2
_<test::Derived2 as test::Base>::foo:
        mov     w8, #3
        mov     x0, x8
        ret
        .p2align        2
_test::call_foo:
        stp     x29, x30, [sp, #-16]!
        mov     x29, sp
        bl      _<test::Derived2 as test::Base>::foo
        ldp     x29, x30, [sp], #16
        ret
        .p2align        2
_test::call_foo:
        stp     x29, x30, [sp, #-16]!
        mov     x29, sp
        bl      _<test::Derived1 as test::Base>::foo
        ldp     x29, x30, [sp], #16
        ret
        .p2align        2
```

So, Rust trait was born to be compile-time polymorphism. It looks much better
than C++ CRTP, right?

Now, let's see runtime polymorphism. The only change we need on top the above
example is below.

```rust
fn call_foo(obj: &dyn Base) -> i64 { obj.foo() }
```

The generated assembly is below.

```
_test2::call_foo:
        stp     x29, x30, [sp, #-16]!
        mov     x29, sp
        ldr     x8, [x1, #24]
        blr     x8
        ldp     x29, x30, [sp], #16
        ret
        .p2align        2
```

There is only one, not two, `call_foo`, so it is not direct dispatching.
Ignoring the function prologue and epilogue, the only code left is

```
ldr     x8, [x1, #24]
blr     x8
```

It loads content at `(X1 + 24)` to register `X8` and then branch to X8, i.e.,
call the function pointer at `X8`. This magic is all about fat pointer and
vtable!

## Fat pointer and Vtable

Trait object is introduced to achieve runtime polymorphism, and similar to C++,
it uses a vtable for runtime dispatching. Note, vtable is just a compiler
implementation detail. It is not a language feature.

We cannot avoid mentioning memory layout when talking about vtable. For
context, the Rust compiler (rustc) determines how values are represented in
memory through its memory layout calculator, which computes each type’s size,
alignment, and field offsets. This information is essential for establishing a
type’s ABI. Once layouts and ABI classifications are determined, rustc lowers
high-level Rust constructs into an intermediate representation (MIR), then
translates them into LLVM IR. LLVM uses the provided layout and ABI details to
generate correct machine code. For trait object, the high-level process is as
follows.

1. The compiler starts with the type `&dyn Trait`. It sees this is a reference
   (`ty::Ref`) and then looks at what it points to: `dyn Trait (ty::Dynamic)`.
   It then computes the layout of `dyn Trait` and discovers that it is unsized.
   It does not have a compile-time constant size.

2. Pointers to Unsized Types Must Be "Fat". This is the critical trigger. The
   rust compiler has a fundamental rule: any pointer to an unsized type must be
   a "fat pointer". It must carry extra information to be useful. For a slice ,
   the extra info is the length. For a trait object, the extra info is a
   pointer to the vtable.

3. Once the compiler knows it needs to create a fat pointer, it knows the
   layout must consist of two pointer-sized components. It calls function
   function
   [LayoutCalculator#scalar_pair](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_ty_utils/src/layout.rs#L297)
   to build this two-pointer layout.

Most code about type memory layout can be found inside
[rustc_ty_utils/src/layout.rs](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_ty_utils/src/layout.rs#L224)
and
[rustc_abi/src/layout.rs](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_abi/src/layout.rs#L92).
For example, see
[here](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_ty_utils/src/layout.rs#L232).
For a reference or raw pointer type, if the pointee is sized, then it just
calculate the memory layout of a single scalar. If not, then it requires a
metadata scalar and then calculate the memory layout of a scalar pair.

Now we understand what it means by a fat pointer. A trait object reference is
lowered to a pair of pointers: `data_ptr` and `vtable_ptr` in memory. For
context, arm64 procedure call standard requires the first few function
arguments are passed to register X0-X7. If the argument is composite and is no
more than 16 bytes, then it can use 2 registers. See my another post "Assembly
-- ARM" for more details. In our case, two pointers take exact 16 bytes:
`data_ptr -> X0` and `vtable_ptr -> X1`. So `ldr x8 [x1, #24]` means loading
the content at `vtable_ptr + 24` to `X8`. We are very close to solving the
puzzle! Similarly to C++, a vtable has many rows. The offset 24 must be the
offset to the dispatched function. Let's check it out.

This is the
[code](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_trait_selection/src/traits/vtable.rs#L250)
that builds vtable. It first adds some common entries: drop_in_place ptr, size
info, alignment info. And then it adds the function pointers of this concrete
subtype.

```
Derived1
+---------------------------------+
| ptr to drop_in_place            |
| size_of::<Derived1>             |
| align_of::<Derived1>            |
| ptr to Derived1::foo            | <--- e.g., foo() is at offset 3, that is 24 bytes.
+---------------------------------+

Derived2
+---------------------------------+
| ptr to drop_in_place            |
| size_of::<Derived2>             |
| align_of::<Derived2>            |
| ptr to Derived2::foo            | <--- e.g., foo() is at offset 3, that is 24 bytes.
+---------------------------------+
```

Finally here!

As a bonus, let's check a function taking a slice as argument.

```rust
#[inline(never)]
fn get_slice_len(s: &[i32]) -> usize {
    s.len()
}
```

The assembly is

```
_test3::get_slice_len:
        mov     x0, x1
        ret
        .p2align        2
```

Now, you understand what happens, right? For a call to the above function, `X0`
stores `data_ptr`. `X1` is the slice length. So `mov x0, x1` means move slice
length to register `X0`. The ABI of arm64 says `X0` is return value register.

## More About Vtable

Vtable can have
[Vacant](https://github.com/rust-lang/rust/blob/d68c32779627fcd72a928c9e89f65094dbcf7482/compiler/rustc_middle/src/ty/vtable.rs#L18)
rows. What does it mean?

dyn upcasting:
[rfc-3324-dyn-upcasting](https://rust-lang.github.io/rfcs/3324-dyn-upcasting.html)

TBD...
