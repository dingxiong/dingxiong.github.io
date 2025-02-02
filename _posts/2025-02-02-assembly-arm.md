---
layout: post
title: Assembly -- ARM
date: 2025-02-02 01:11 -0800
categories: [assembly]
tags: [assembly, arm, arm64, aarch64]
---

Aarch64 is just arm64. The instruction set used in aarch64 is called A64.

## Prologue and Epilogue

(TODO: below part is wrong. Fix it.)

We usually see two different flavors of prologue and epilogue on internet

```
sub sp, sp, #16          // Allocate 16 bytes for frame record
stp x29, x30, [sp]      // Save x29 (FP) and x30 (LR) to stack
mov x29, sp             // Set FP (frame pointer) to current SP

ldp x29, x30, [sp]      // x29 = [sp]; x30 = [sp + 8]
add sp, sp, #16
ret
```

and

```
sub sp, sp, #32         // Allocate 32 bytes for stack frame
stp x29, x30, [sp, #16] // Save x29 (FP) and x30 (LR) at offset 16
add x29, sp, #16        // Set FP to sp + 16 (to point to saved x29/x30)

ldp x29, x30, [sp, #16]
add sp, sp, #32
ret
```

The former is a basic Stack Frame Setup. The latter is an advanced Frame
Layout. Why called advanced? Because the stack portion [sp, sp+16] can be used
for local variables. Why only reserving 16 bytes for local variables? It is a
balance. For a function using a lot of local variables, 16 bytes are not
enough. But for functions that do not use local variables, it is a waste of
stack space. 16 bytes strikes a balance.

See [reference](https://johannst.github.io/notes/arch/arm64.html).

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
