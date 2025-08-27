---
layout: post
title: Assembly -- ARM
date: 2025-02-02 01:11 -0800
categories: [compiler, assembly]
tags: [assembly, arm, arm64, aarch64]
---

Aarch64 is just arm64. The instruction set used in aarch64 is called A64.

## Procedure Call Standard (PCS)

The
[ARM standard](https://developer.arm.com/documentation/102374/0102/Procedure-Call-Standard)
has a clear description of PCS.

1. The first few parameters should be loaded to X0-X7. If more than 8
   arguments, then the rest should be be put to stack.
2. If the parameter is composite, i.e., a struct in C, then depending on the
   size, there are different conventions. Larger than 16 bytes, then the
   argument is copied to memory allocated by the caller and the argument is
   replaced by a pointer to the copy. Otherwise, rounding up to the nearest
   multiple of 8 bytes and put them inside registers directly. The
   [abi documentation section 6.8.2](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst#parameter-passing-rules)
   mentions the rules but as a reader, I feel the details are so fragmented.
   Let's take a look at a few examples.
   - `struct {int32 a, int32 b}`: a → lower 32 bits of X0, b → upper 32 bits of
     x0
   - `struct {int32 a, int64 b}`: X0 = [padding(upper 32 bits) | a (lower 32
     bits)]; X1 = b. We need to put b to X1 instead of spreading it between X0
     and X1 because int64 requires 8 bytes padding.
3. The return values is stored in X0, X0/X1 or X8 depending on the size of the
   return value. Most functions return a single value (X0). Small structs (≤16
   bytes) can be returned using X0 and X1. If the return struct is larger than
   16 bytes. the caller will allocated some memory on stack and put the pointer
   inside X8, and the callee will populate the content pointed by this pointer.
   See the
   [documentation](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst#result-return)

I learned two things. First, I learned that a C function can return a struct. I
thought it can only return primitive or pointer types. Second, Aarch64 can use
both X0 and X1 for return value. This fact confused me because why only X0, X1,
but not extend to X2, X3 etc as well. I do not find any official documentation
confirming it, and lost in LLVM backend code. However, LLVM can quickly
generate C programs simulating all these scenarios, and from the assembly
generated, I can confirm that this is true: as long as struct size goes beyond
16 bytes, the caller and callee uses X8 to communicate return value.

There is a noticeable difference for variadic functions in MacOS. The
[MacOS ARM64 ABI](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms#Update-code-that-passes-arguments-to-variadic-functions)
specifies that variadic arguments should be stored on stack as well, and should
be 8-byte stack aligned. For example, for a call
`int x, y; printf("msg %d %d\n", x, y);`, `X0` should be the address of "msg
%d\n". `X1` and `X2` should be `x` and `y`. In addition, stack should also have
`x` and `y` and each takes 8 bytes.

## Prologue and Epilogue

I tried to ask chatgpt/claude/deepseek about ARM64 prologue and epilogue, but
all of them generated wrong results. Then I wrote a small C program.

```
#include <stdio.h>

long add(long x1, long x2, long x3, long x4, long x5, long x6) {
  return x1 + x2 + x3 + x4 + x5 + x6;
}

int main() {
    long x1 = 1;
    long x2 = 2;
    long x3 = 3;
    long x4 = 4;
    long x5 = 5;
    long x6 = 6;
    printf("%dl\n", add(x1, x2, x3, x4, x5, x6));
    return 0;
}
```

The generated assembly is

```
...
_main:                                  ; @main
        sub     sp, sp, #96
        stp     x29, x30, [sp, #80]             ; 16-byte Folded Spill
        add     x29, sp, #80
        mov     w8, #0                          ; =0x0
        str     w8, [sp, #20]                   ; 4-byte Folded Spill
        stur    wzr, [x29, #-4]
        mov     x8, #1                          ; =0x1
        stur    x8, [x29, #-16]
        mov     x8, #2                          ; =0x2
        stur    x8, [x29, #-24]
        mov     x8, #3                          ; =0x3
        stur    x8, [x29, #-32]
        mov     x8, #4                          ; =0x4
        str     x8, [sp, #40]
        mov     x8, #5                          ; =0x5
        str     x8, [sp, #32]
        mov     x8, #6                          ; =0x6
        str     x8, [sp, #24]
        ldur    x0, [x29, #-16]
        ldur    x1, [x29, #-24]
        ldur    x2, [x29, #-32]
        ldr     x3, [sp, #40]
        ldr     x4, [sp, #32]
        ldr     x5, [sp, #24]
        bl      _add
        mov     x8, sp
        str     x0, [x8]
        adrp    x0, l_.str@PAGE
        add     x0, x0, l_.str@PAGEOFF
        bl      _printf
        ldr     w0, [sp, #20]                   ; 4-byte Folded Reload
        ldp     x29, x30, [sp, #80]             ; 16-byte Folded Reload
        add     sp, sp, #96
        ret
.section  __TEXT,__cstring,cstring_literals
      l_.str:                                 ; @.str
        .asciz  "%dl\n"
...
```

First, the output is totally unoptimized and even wasteful. There are only 6
local variables, so 48 (6\*8) bytes together with 16 bytes for x29 and x30
should be enough. We do not need 96 bytes. Let's use `sp'` denoting the `sp`
before `sub sp, sp, #96`. The memory layout is

```
high memory address
+------------------+  <-- Old SP (before allocation)
|   Previous Frame |
|------------------|
|   Saved x30 (LR) |  <-- sp + 88
|   Saved x29 (FP) |  <-- sp + 80  = x29
|   Local Var x1   |  <-- x29 - 4
|   Local Var x2   |  <-- x29 - 16
|   Local Var x3   |  <-- x29 - 24
|   Local Var x4   |  <-- x29 - 32
|   Local Var x5   |  <-- x29 - 40 = sp + 40
|   Local Var x6   |  <-- x29 - 48 = sp + 32
|    ...           |
|                  |  <-- x29 - 80 = sp
+------------------+
low memory address
```

We can generalize the pattern. Suppose we have `N` local variables and each
takes 8 bytes, then the function prologue is

```
sub     sp, sp, #(1+N/2)*16         ; grow stack for (1+N/2)*16 bytes. sp must be 16 bytes aligned.
stp     x29, x30, [sp, #N/2*16]     ; store x29 and x30 at the higher memory address, so local variables grow toward to the top of stack.
add     x29, sp, #N/2*16            ; set the new frame pointer x29 to point to the location which is just above the first local variable.
```

and epilogue is

```
ldp     x29, x30, [sp, #N/2*16]     ; restore frame pointer and link register.
add     sp, sp, #(1+N/2)*16         ; resort stack pointer
ret
```

There are many offset calculation. I find that we can simply epilogue a little
bit. `x29` won't change inside this function, and assume we do not change `sp`
too. A child function call inside the current function maintains invariance of
`sp`, so this assumption is pretty valid. So `x29 = sp + N/2 * 16` and thus

```
mov sp, x29             ; x29 = sp + N/2 * 16, so this is equivalent to `add sp, sp, N/2*16`
ldp x29, x30, [sp]        ; equivalent to `ldp x29, x30, [old_sp, #N/2*16]`;
add sp, sp, #16         ; totally restore sp.
ret
```

TODO: need to verify this actually works.

When there is no local variables (N = 0), we can simplify it to

```
stp x29, x30, [sp, #-16]!   ; Pre-decrement SP and store FP and LR
mov x29, sp                 ; Set up frame pointer

ldp x29, x30, [sp], #16    // Restore FP and LR, post-increment SP
ret
```

See [reference](https://johannst.github.io/notes/arch/arm64.html).

## MOV and LDR

In arm64, `mov` and `ldr` serve different purposes. `mov` copies data between
registers, or to load an immediate value directly into a register. `ldr` loads
data from memory into a register. This is dramatically different from x86_64,
where `mov` is used for all all kinds of data copy. `mov` in x86_64 is like the
combination of `mov` and `ldr` in arm64 and is more than that.

## Load data by label

LDR is also a pseudo-instruction. The syntax is `LDR Xd, =label`. It is so
convenient to use it to load the address of a label or constant to a register.
However, it has two drawbacks. First, it only works for 4MB address space.
Second, it uses literal pool which incurs memory reference. Moreover, it does
not work inside MacOS ARM64. In MacOS, you should use below style instead.

```
.data
my_number: .byte 5

.text
adrp    x0, my_number@PAGE      // Get high bits of address
add     x0, x0, my_number@PAGEOFF // Add low bits of offset
```

The `@PAGE` and `@PAGEOFF` assembler expressions are specific to Mach-O
relocations on macOS ARM64.
[asm book](https://github.com/pkivolowitz/asm_book/blob/main/section_1/regs/ldr2.md#apple-silicon)
has a good explanation about it.
