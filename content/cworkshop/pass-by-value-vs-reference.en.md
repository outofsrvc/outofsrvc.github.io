---
title: "Passing by Value vs by Reference"
description: "Value vs reference passing fully traced in x86-64 assembly — the golden rule for telling mov apart from lea."
date: 2026-08-13T10:00:00+03:00
slug: "pass-by-value-vs-reference"
weight: 14
hex: "0x13"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "assembly", "x86-64", "pointers"]
translationKey: "pass-by-value-vs-reference"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

From this article we begin entering **x86-64 Assembly** to gain low-level understanding. The explanation will be simple and clear.

> **Prerequisite**: a study of computer architecture and organization, plus basic knowledge of memory and registers (covered in the pointers and stack-vs-heap articles), is recommended before continuing this series.

> **Example environment**: the gcc outputs here are from Linux (the SysV convention, where the first argument goes in RDI). On Windows x64 the argument registers differ (RCX/RDX/R8/R9) as explained in the virtual memory article.

## Introduction to Assembly

Assembly is a low-level programming language (abbreviated asm) — called low-level because it is **the closest to the CPU** of all languages (C/C++, Java...), and it is **the first human-readable layer** above the code the processor executes.

```text
C/C++  →  Assembly  →  Machine Code (Hex)  →  Binary  →  CPU
```

Every instruction in asm is a **mnemonic**, translated into an **opcode** (a hex representation of the machine instruction), which then becomes binary the CPU executes.

> **Note**: the semicolon `;` in asm marks a comment.

### The Essential Mnemonics (~90% of asm code)

**1. Data movement:**

```asm
mov  destination, source  ; move data from source to destination (like x = 5)
```

**2. Arithmetic:**

```asm
add  dest, src  ; dest = dest + src
sub  dest, src  ; dest = dest - src
inc  operand    ; operand++
dec  operand    ; operand--
```

**3. Comparison and control:**

```asm
cmp  op1, op2   ; compare op1 and op2
jmp  label      ; unconditional jump to label
```

**4. Stack management:**

```asm
push value   ; place a value on the stack
pop  value   ; retrieve a value from the stack
call func    ; call a function (pushes the return address)
ret          ; return from a function to the return address
```

## Passing Arguments in C

Arguments reach functions in two main ways:
- **Pass by Value**
- **Pass by Reference** (via pointers)

## 1. Pass by Value

When an argument is passed by value, the function creates a **copy** of the original variable — any change inside the function **does not affect** the original.

```c
#include <stdio.h>

void modify(int x) {
    x = 20; // modifies only the local copy
    printf("inside function: %d\n", x); // 20
}

int main() {
    int num = 10;
    modify(num); // pass by value
    printf("outside function: %d\n", num); // 10 (unchanged)
    return 0;
}
```

### Memory Analysis

1. `num` in `main()` sits at hypothetical address `0x1000` holding 10.
2. When `modify(num)` is called a new copy `x` is created (say at `0x2000`) holding 10.
3. The assignment `x = 20` changes only address `0x2000`.
4. The original `num` stays at `0x1000` unchanged.

### Turning C into ASM

Using gcc:

```bash
gcc -S -masm=intel -fno-asynchronous-unwind-tables -fno-pie -O0 file.c -o file.s
```

> - `-masm=intel`: Intel syntax (the easier one to read).
> - `-fno-asynchronous-unwind-tables`: removes `.cfi` directive noise.
> - `-fno-pie`: simplifies addressing.
> - `-O0`: no optimizations — the clearest code.

The generated code:

```asm
.LC0:
    .string    "inside function: %d\n"
    .text
    .globl    modify
    .type    modify, @function
modify:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     DWORD PTR [rbp-4], edi   ; the copy x = first argument (edi)
    mov     DWORD PTR [rbp-4], 20    ; x = 20 (the copy only)
    mov     eax, DWORD PTR [rbp-4]
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC0
    mov     eax, 0
    call    printf
    nop
    leave
    ret
    .size    modify, .-modify

    .section    .rodata
.LC1:
    .string    "outside function: %d\n"
    .text
    .globl    main
    .type    main, @function
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     DWORD PTR [rbp-4], 10    ; num = 10
    mov     eax, DWORD PTR [rbp-4]   ; eax = num
    mov     edi, eax                 ; first argument = 10 (the value!)
    call    modify
    mov     eax, DWORD PTR [rbp-4]   ; num is still 10
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC1
    mov     eax, 0
    call    printf
    mov     eax, 0
    leave
    ret
```

### Explaining the Assembly (Pass by Value)

**In `main`:**

```asm
main:
    sub     rsp, 16                    ; reserve 16 bytes on the stack
    mov     DWORD PTR -4[rbp], 10      ; [rbp-4] = num = 10
    mov     eax, DWORD PTR -4[rbp]     ; eax = num's value (10)
    mov     edi, eax                   ; copy 10 into edi
    call    modify                     ; call modify
```

**In `modify`:**

```asm
modify:
    mov     DWORD PTR -4[rbp], edi     ; [rbp-4] = edi = 10 (a fresh copy, distinct from num)
    mov     DWORD PTR -4[rbp], 20      ; change the copy to 20 — num stays untouched
```

**Back in `main`:**

```asm
    mov     eax, DWORD PTR -4[rbp]     ; eax = num (still 10)
```

> **The golden rule**: with pass by value you see `mov` moving **the value** into a register (`mov edi, eax`) — the value itself is copied.

## What Is DWORD PTR?

A size directive — it specifies the size of the data an instruction operates on, because **the processor does not automatically know** how large the data it reads/writes from memory is.

| Term | Size |
|---|---|
| **BYTE** | 1 byte = 8 bits |
| **WORD** | 2 bytes = 16 bits |
| **DWORD** | 4 bytes = 32 bits |
| **QWORD** | 8 bytes = 64 bits |

**PTR = Pointer** — indicating we are dealing with a memory address.

## 2. Pass by Reference

C has **no reference variables** like C++ — so we use **pointers**. We pass the variable's address, and modifications hit the original directly.

```c
#include <stdio.h>

void modify(int *x) {
    *x = 20; // modifies the original variable
    printf("inside function:%d\n", *x); // 20
}

int main() {
    int num = 10;
    modify(&num); // pass the address
    printf("outside function:%d\n", num); // changed to 20
    return 0;
}
```

### Memory Analysis

1. `num` holds 10 at address `0x1000`.
2. When `modify(&num)` is called, address `0x1000` is passed into the pointer `x`.
3. The assignment `*x = 20` changes the value at `0x1000` directly.
4. The original `num` becomes 20.

### The Assembly (Pass by Reference)

```asm
.LC0:
    .string    "inside function:%d\n"
    .text
    .globl    modify
    .type    modify, @function
modify:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     QWORD PTR [rbp-8], rdi   ; save the address (an 8-byte pointer)
    mov     rax, QWORD PTR [rbp-8]   ; rax = the original address
    mov     DWORD PTR [rax], 20      ; *x = 20 (directly at the original address)
    mov     rax, QWORD PTR [rbp-8]
    mov     eax, DWORD PTR [rax]
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC0
    mov     eax, 0
    call    printf
    nop
    leave
    ret
    .size    modify, .-modify

    .section    .rodata
.LC1:
    .string    "outside function:%d\n"
    .text
    .globl    main
    .type    main, @function
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     rax, QWORD PTR fs:40      ; stack canary setup begins
    mov     QWORD PTR [rbp-8], rax
    xor     eax, eax
    mov     DWORD PTR [rbp-12], 10    ; num = 10
    lea     rax, [rbp-12]             ; rax = num's address (&num)
    mov     rdi, rax                  ; rdi = the address (passed as the argument)
    call    modify
    mov     eax, DWORD PTR [rbp-12]   ; eax = num (now 20)
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC1
    mov     eax, 0
    call    printf
    mov     eax, 0
    mov     rdx, QWORD PTR [rbp-8]    ; canary verification
    sub     rdx, QWORD PTR fs:40
    je      .L4
    call    __stack_chk_fail          ; if the canary changed ← failure
.L4:
    leave
    ret
```

### Explaining the Assembly (Pass by Reference)

**In `main`:**

```asm
    mov     DWORD PTR -12[rbp], 10   ; [rbp-12] = num = 10
    lea     rax, -12[rbp]            ; rax = num's address (&num)
    mov     rdi, rax                 ; rdi = the address (passed as argument)
    call    modify
```

**In `modify`:**

```asm
    mov     QWORD PTR -8[rbp], rdi   ; saves the original address in a local
    mov     rax, QWORD PTR -8[rbp]   ; rax = the original address
    mov     DWORD PTR [rax], 20      ; *rax = 20 — directly at the original address
```

**Back in `main`:**

```asm
    mov     eax, DWORD PTR -12[rbp]  ; eax = num (20) — the new value
```

> **The golden rule**: with pass by reference you see `lea` **computing an address**, then `mov` into a 64-bit register (`mov rdi, rax`) — the **address** is passed, not the value.

## The New Instruction: `lea`

**Load Effective Address** — computes the effective address of a memory operand and stores it in the destination **without touching memory**.

```asm
lea rax, [rbp-12]   ; rax = &num — address computation only, no memory read
```

> **The fundamental difference**: `mov eax, [rbp-4]` reads **the value** from memory, while `lea rax, [rbp-12]` computes **the address** only. That is why seeing `lea` in a disassembly tells you the code was passing by reference.

## A Special Case: Passing Arrays

Arrays are passed **automatically by reference** — the array name decays into a pointer to its first element.

```c
#include <stdio.h>

void changeArray(int arr[]) {
    arr[0] = 100; // modifies the original array directly
}

int main() {
    int a[3] = {1, 2, 3};
    changeArray(a); // pass the address without &
    printf("%d\n", a[0]); // 100
    return 0;
}
```

## An Important Note: The Pointer Itself Is a Copy

When you pass a pointer, a **copy of the address** is passed — you can modify what it points to (`*ptr = value`), but if you change the pointer itself (`ptr = &new`) the original pointer outside the function is unaffected. To change the original pointer we use a **pointer to pointer** (`int **ptr`):

```c
#include <stdio.h>
#include <stdlib.h>

void changePointer(int **ptrToPtr) {
    int *new_ptr = malloc(sizeof(int));
    *new_ptr = 100;
    *ptrToPtr = new_ptr; // make the original pointer point at the new block
}

int main() {
    int x = 5;
    int *org_ptr = &x;

    printf("pointer before change: %d\n", *org_ptr); // 5

    changePointer(&org_ptr); // pass the pointer's address

    printf("pointer after change: %d\n", *org_ptr); // 100

    free(org_ptr);
    return 0;
}
```

## Important Questions (Answered)

**1. How do we recognize pass by reference?**
By the presence of `lea` — it computes addresses without touching memory. The pattern `lea` + `mov rdi, rax` = pass by reference.

**2. Why did we use `rax` with by-reference and `eax` with by-value?**
`eax` is the lower 32-bit half of `rax`. Addresses/pointers use full 64-bit registers because **an address is 8 bytes** — using 32-bit would truncate it (4 bytes only).
> Extra detail: on x64, writing to a 32-bit register (like `mov eax, ...`) **automatically zeroes the upper half** of the 64-bit register — important behavior when analyzing code.

**3. Why `[rbp-12]` in by-reference but `[rbp-4]` in by-value?**
Because the by-reference example includes a **stack canary** — a random value (8 bytes) placed between locals and the return address to protect it from buffer overflows:
- GCC adds the canary automatically when **the address of a local variable is passed to another function** (exactly what happens with by-reference).
- At function end the canary is verified; if it changed (a buffer was overrun) it calls `__stack_chk_fail` and stops the program.

Stack layout:

```text
by value:                 by reference:
+-----------------+       +-----------------+
|  num  [rbp-4]   |       |  num  [rbp-12]  |
+-----------------+       |  canary [rbp-8] |
| return address  |       +-----------------+
+-----------------+       | return address  |
                          +-----------------+
```

## Why Bring Assembly into a C Series?

To reach the advanced levels of reverse engineering you must **understand code line by line at the low level**. Learning hands-on while watching C turn into assembly — and understanding how asm moves with C — is the most effective way to start.

## The RE Connection

1. **The `lea` = by-reference rule**: when analyzing any binary, tracing `lea` patterns reveals arguments passed by reference — helping you interpret parameters in system functions and APIs.
2. **The first argument in RDI/EDI** (SysV x86-64): an iron rule — the first argument is always in RDI/EDI, making any disassembly far faster to read. On Windows x64 it is RCX instead — see the x64 convention in the virtual memory article.
3. **Stack canary**: seeing `fs:40` patterns in a disassembly means the code was compiled with buffer overflow protection — which determines in advance the kind of challenge an exploiter faces.

> To read code at the low level, write small C programs and review their `gcc -S` output — it is the fastest way to build intuition. And peace be upon you and the mercy of Allah and His blessings.











