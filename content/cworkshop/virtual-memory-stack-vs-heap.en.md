---
title: "Virtual Memory: Stack vs Heap"
description: "Stack frames, registers, and calling conventions, plus heap chunks — the two primary battlegrounds of exploitation."
date: 2026-08-13T10:00:00+03:00
slug: "virtual-memory-stack-vs-heap"
weight: 12
hex: "0x11"
stage: "memory"
categories: [c-lang]
tags: ["c-lang", "memory", "stack", "heap", "calling-conventions", "assembly"]
translationKey: "virtual-memory-stack-vs-heap"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we discuss the design of stack and heap memory, and clarify the differences between them.

> **Note**: this article does not cover everything exhaustively, but it will bring us to a full understanding of the topics we discuss.

## Memory Layout

As mentioned in the pointers article, program memory in modern x86/x64 systems divides into 5 main segments (some say 4 by merging Data with BSS):

1. **Text Segment** — the executable instructions (read-only).
2. **Data Segment** — initialized globals.
3. **BSS Segment** — uninitialized globals.
4. **Heap** — dynamic allocation.
5. **Stack** — locals and function frames.

```text
+-----------------+  high addresses
|      Stack      |  grows downward
|        |        |
|        v        |
+-----------------+
|      (gap)      |
+-----------------+
|        ^        |
|        |        |
|      Heap       |  grows upward
+-----------------+
|      BSS        |
|     Data        |
|     Text        |
+-----------------+  low addresses
```

> We are dealing only with the **user space** here — kernel space is another topic.

**Why the opposite growth directions**: so that the stack and heap can grow **without colliding** — each expands in a direction away from the other.

## Why Is the Stack Faster? The Secret Is in the Registers

The stack is organized automatically by the processor, which makes it fast. The secret is the **direct relationship between the stack and the registers** — unlike the heap, which interacts with them indirectly.

### The Key Registers

Registers are small storage units inside the processor used for computations requiring high speed:

| Register | Full name | Purpose |
|---|---|---|
| **EAX** | Accumulator | holding function return values, and arithmetic |
| **EBP** | Base Pointer | address of the current stack frame's base |
| **ESP** | Stack Pointer | address of the top of the stack |
| **EIP** | Instruction Pointer | address of the next instruction the processor will execute |
| **ECX** | Counter | counter for repetitive operations (like loops) |
| **EDX** | Data | EAX's partner in multiplication/division; general data |
| **EBX** | Base | no fixed role — general data across contexts |
| **ESI/EDI** | Source/Destination | string transfers: ESI = source, EDI = destination |

> These are x86 (32-bit) names. On x64 they become (RAX, RBP, RSP, RIP...) at 64 bits wide. Our examples use x86.

## What Is the Stack?

A region of memory (logically, not physically separate) that stores local variables and stack frame data. It is reserved automatically via `push`/`pop` as functions are entered and discarded when they return. It follows the **LIFO** pattern — variables created first are removed last.

### The Stack Frame (Activation Record)

A data structure created on the stack when a function is called and destroyed when it returns. Its goal is to **isolate each function from the others** — every call creates its own frame. Components:

1. **Parameters** — the values passed into the function.
2. **Return Address** — where the program resumes after the function ends.
3. **Frame Pointer** — the base of the current stack frame.
4. **Local Variables** — the current function's variables.
5. **Saved Registers** — the caller's register values preserved before calling another function.

### The x86 Stack Frame Layout

```text
[esp]        ← top of stack (ESP)
[ebp-4]      → first local variable
[ebp]        → saved EBP (caller's frame)
[ebp+4]      → return address
[ebp+8]      → first argument
[ebp+12]     → second argument
```

> **Why do we subtract when adding to the stack?** Because the stack grows **downward**: if `ebp = 1000`, the first local sits at `996`, the second at `992`, and so on (in a 32-bit system).

### How a Stack Frame Is Created (Function Prologue)

```c
#include <stdio.h>

int add(int a, int b){
    int result = a + b;
    return result;
}

int main(){
    int x = 5, y = 3;
    add(x, y);
    return 0;
}
```

In detail:
- First `main()` is created with a frame holding its variables `x, y`.
- When `add()` is called, `main()`'s execution pauses and a frame for `add()` is built on top of `main()`'s containing: the parameters (`b=3, a=5`), the local `result`, and the return address.
- When `add()` finishes, its frame is deleted and execution returns to `main()` from the return address.

In assembly:

```asm
; 1. Passing parameters (right to left)
push 3    ; b = 3
push 5    ; a = 5

; 2. Calling the function (places the return address automatically)
call add  ; push EIP + jmp to add

; 3. Prologue
add:
    push ebp        ; save the caller's frame (caller's ebp)
    mov ebp, esp    ; establish a new frame
    sub esp, 4      ; reserve space for the local result
```

### How a Stack Frame Is Destroyed (Function Epilogue)

```asm
; 1. Storing the result in eax
mov eax, [ebp-4]    ; eax = result

; 2. The epilogue
mov esp, ebp        ; restore esp to ebp's position (wipes the locals)
pop ebp             ; restore the caller's frame
ret                 ; pop the return address and jump to it
```

## Calling Conventions

A set of rules agreed between compiler and assembler that define:
1. How parameters are passed to functions.
2. How return values are delivered.
3. Who is responsible for cleaning the stack after a call.
4. Which registers must be preserved.

**Why different patterns exist?** For performance, for interoperability with other systems and languages, and to support special cases (like variadic functions).

### Register Preservation (Important)

| Kind | Registers | Preserved by |
|---|---|---|
| **Caller-saved** | EAX, ECX, EDX | the caller, if it needs them after the call |
| **Callee-saved** | EBX, ESI, EDI, EBP, ESP | the callee must save and restore them |

### 1. cdecl (C Declaration)

| Point | Rule |
|---|---|
| Argument order | right to left |
| Return value | in EAX |
| Stack cleanup | the **caller** |
| Registers | caller saves caller-saved; callee saves callee-saved |

```c
int __attribute__((cdecl)) add(int a, int b){
    return a + b;
}

int result = add(5, 3);
```

```asm
; executed by the caller
push 3        ; b = 3
push 5        ; a = 5
call _add
add esp, 8    ; stack cleanup — the caller is responsible
```

### 2. stdcall (Standard Call)

| Point | Rule |
|---|---|
| Argument order | right to left |
| Return value | in EAX |
| Stack cleanup | the **callee** |
| Registers | the callee preserves EBP, EBX, EDI, ESI, ESP |

```c
int __attribute__((stdcall)) add(int a, int b){
    return a + b;
}
```

```asm
; the caller
push 3        ; b = 3
push 5        ; a = 5
call _add@8   ; the name encodes the byte count (@8)

; inside the function (the callee)
ret 8         ; clean the stack and return
```

### 3. fastcall (Fast Call)

| Point | Rule |
|---|---|
| Arguments | first two in ECX, EDX; the rest on the stack (right→left) |
| Return value | in EAX |
| Stack cleanup | the callee |
| Registers | the callee preserves the registers it uses |

```c
int __attribute__((fastcall)) add(int a, int b, int c){
    return a + b + c;
}

int result = add(3, 5, 2);   // a=3, b=5, c=2
```

```asm
; the caller
mov ecx, 3     ; a = 3 in ECX
mov edx, 5     ; b = 5 in EDX
push 2         ; c = 2 on the stack
call @add@12

; inside the function (the callee)
ret 4          ; clean the stack (argument c only)
```

### 4. The x64 Convention (Modern Architectures)

> **Very important for your blog's focus (Windows)**: on 64-bit there is no cdecl/stdcall/fastcall. There is a single convention:
> - The first four arguments go in: **RCX, RDX, R8, R9**; the rest go on the stack.
> - A **shadow space** (32 bytes) must be reserved on the stack.
> - The caller cleans the stack (stack-passed arguments only).
> - Return values are in **RAX**.
>
> When analyzing modern binaries, this is what you will use most of the time.

### How to Choose the Right Pattern?

- **cdecl**: for variadic functions (like `printf`).
- **stdcall**: for the Windows API.
- **fastcall**: when speed matters (uses ECX/EDX registers).

## The Heap

Not a separate memory region but part of the process address space. Unlike the stack (managed automatically), the heap is **managed manually by the programmer** and used for runtime allocation.

### Heap Properties

- Variable size (grows and shrinks as needed).
- Allocations persist until manually released with `free()`.
- Far larger than the stack.

### The Fundamental Unit: The Chunk

Every allocation in the heap (via `malloc` and friends) is stored in a unit called a **chunk** — even if you request a tiny space (`malloc(1)`), a full chunk is allocated.

### Chunk Structure

```text
+-------------------+
|   mchunk_size     |  total chunk size (including the header)
|   + flags         |  3 information bits
+-------------------+
|   User data       |  the space actually handed to the user
+-------------------+
|   padding         |  filler to round the size
+-------------------+
```

**The header** (16 bytes on 64-bit):
- **mchunk_size**: the chunk's total size.
- **flags** — 3 bits:
  - **Bit 0 (PREV_INUSE)**: `1` = previous chunk in use, `0` = free.
  - **Bit 1 (IS_MMAPPED)**: `1` = allocated via mmap, `0` = via sbrk.
  - **Bit 2 (NON_MAIN_ARENA)**: `1` = from a secondary arena (multi-threading), `0` = main arena.

**If the chunk is free**, extra information is stored in the header:
- **fd**: pointer to the next free chunk.
- **bk**: pointer to the previous free chunk.

### The Two Heap Growth Mechanisms: sbrk and mmap

| | **sbrk()** | **mmap()** |
|---|---|---|
| **Mechanism** | moves the program break pointer upward | creates a separate memory region outside the main heap |
| **Release** | can only shrink from the top | via `munmap(ptr, size)` |
| **Limits** | cannot release middle portions; unsuitable for many threads | suitable for large blocks |
| **When** | the default | for blocks larger than **128KB** |

### Structural Transformations of the Heap

1. **On allocation**: the chunk is in use (PREV_INUSE set).
2. **On free**: the chunk becomes free; fd/bk are initialized and it is linked into the free list.
3. **Coalescing**: two adjacent free chunks merge into one giant free chunk.

### Computing Chunk Size (64-bit System)

```text
Total size = the nearest multiple of 16 of (header size + requested size)
Padding = total size − (header size + requested size)
```

**Examples:**
- `malloc(1)`: 16 + 1 = 17 → nearest multiple of 16 = **32** → padding = 32 - 17 = **15 bytes**.
- `malloc(100)`: 100 + 16 = 116 → nearest multiple of 16 = **128** → padding = 128 - 116 = **12 bytes**.

**Important notes:**
- The minimum total chunk size is **32 bytes**, even for `malloc(0)`.
- Chunks larger than 128KB are allocated via mmap and usually have a different header (32 bytes).

## Stack vs Heap

| | **Stack** | **Heap** |
|---|---|---|
| **Management** | automatic (the processor) | manual (the programmer) |
| **Growth** | from high addresses downward | from lower addresses upward |
| **Speed** | fast (direct register relationship) | slower |
| **Size** | limited | large |
| **Storage model** | LIFO | free-form |
| **Fundamental unit** | Stack Frame | Chunk |
| **Lifetime** | until the function returns | until `free()` |
| **Danger** | Buffer Overflow (over the return address) | UAF / Double Free / Heap Overflow |

## The RE Connection — Why This Article Is a Treasure Trove

1. **The stack frame is the foundation of buffer overflow**: the return address (`[ebp+4]`) is **the most famous exploitation target** — an overflowing write over it (via a buffer overflow) redirects execution to shellcode or a ROP chain. Everything explained here is the foundation the coming exploitation content builds on.

2. **The chunk is the foundation of heap exploitation**: fd/bk in free chunks are what attacks like fastbin/tcache poisoning abuse. The full heap exploitation series will be built on this article.

3. **Calling conventions separate real analysts**: seeing `add esp, 8` after a call = cdecl; its absence = stdcall. This difference instantly identifies the kind of code you are analyzing.

4. **sbrk/mmap**: when you see a huge chunk during malloc internals analysis, you know it was mmap'd — a small detail that marks the meticulous analyst.

## Summary

The stack and heap are the **two primary battlegrounds** of exploitation. Understanding their anatomy (the stack frame with its return address, and the chunk with its fd/bk) is what will let you understand any memory vulnerability you meet in future articles — from Buffer Overflow to Heap Exploitation.










