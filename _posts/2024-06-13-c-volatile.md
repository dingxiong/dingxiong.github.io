---
layout: post
title: C -- volatile
date: 2024-06-13 23:34 -0700
categories: [programming-language, c]
tags: [volatile]
---

When using GCC to add breakpoint to a specific location, sometimes I am worried
that this piece of code may be optimized away by the compiler. Disabling
optimization can rule out my concern, but I also wonder whether there is a
language semantic that prohibits a line of code being optimized away. The
answer is [volatile](https://barrgroup.com/blog/how-use-cs-volatile-keyword).

Let's do a simple experiment.

```c
/* test.c */
int main() {
  int meaning_of_life = 42;
  return 0;
}
```

`gcc -S test.c -03` generates below assembly in MacOS M1.

```asm
        .section        __TEXT,__text,regular,pure_instructions
        .build_version macos, 14, 0     sdk_version 14, 5
        .globl  _main                           ; -- Begin function main
        .p2align        2
_main:                                  ; @main
        .cfi_startproc
; %bb.0:
        mov     w0, #0
        ret
        .cfi_endproc
                                        ; -- End function
.subsections_via_symbols
```

`mov w0 #0` loads intermediate 0 to register `w0` which is used to set the
return value of the function. As expected, `int meaning_of_life = 1;` is
optimized away. However, if we change this line to

```
volatile int meaning_of_life = 42;
```

we get below assembly

```asm
        .section        __TEXT,__text,regular,pure_instructions
        .build_version macos, 14, 0     sdk_version 14, 5
        .globl  _main                           ; -- Begin function main
        .p2align        2
_main:                                  ; @main
        .cfi_startproc
; %bb.0:
        sub     sp, sp, #16
        .cfi_def_cfa_offset 16
        mov     w8, #42
        str     w8, [sp, #12]
        mov     w0, #0
        add     sp, sp, #16
        ret
        .cfi_endproc
                                        ; -- End function
.subsections_via_symbols
```

Ignore the function prologue and epilogue. `mov w8 #42` loads intermediate
value 42 to register `w8`. `str w8, [sp, #12]` stores the value in register w8
into the stack at an offset of 12 bytes from the current stack pointer, which
is the location of local variable `meaning_of_life` in the stack.

OK. A `volatile` variable indeed prevents compiler optimizing it away.
