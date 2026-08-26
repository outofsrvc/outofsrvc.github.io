---
title: "Functions"
description: "Function types, the memory and I/O functions that matter to analysts, plus prologue/epilogue structure and calling conventions."
date: 2026-08-13T10:00:00+03:00
slug: "functions"
weight: 9
hex: "0x08"
stage: "memory"
categories: [c-lang]
tags: ["c-lang", "assembly", "stack", "basics"]
translationKey: "functions"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will cover functions, their types and forms, and the functions that matter most in reverse engineering.

**Functions play a pivotal role in programming, and especially in reverse engineering** — they are the fundamental unit of analysis: when you open any program in IDA or Ghidra, the first thing you see is a list of functions, and understanding them is how you understand the program.

## What Is a Function?

Fundamental units for reusing and organizing code — "blocks" of instructions designed to perform specific tasks. Every function can accept arguments and return a value.

```c
return_type function_name(parameter_list) {
    // body: the code that performs the task
}
```

### A Simple Example

```c
int add(int a, int b) {
    return a + b;
}

int main() {
    int result = add(5, 3);
    printf("The sum is: %d\n", result);
    return 0;
}
```

> **Important note**: a function does not execute by itself — it must be called from another function. `main` is where program execution starts, and calls spread out from it to the rest of the functions.

## Function Forms

### 1. No Arguments, No Return

```c
void hello() {
    printf("Hello, World!\n");
}

int main() {
    hello();
    return 0;
}
```

### 2. With Arguments, No Return

```c
void displayNumber(int number) {
    printf("The number is: %d\n", number);
}

int main() {
    displayNumber(10);
    return 0;
}
```

> Quick rule: a function that returns nothing is declared `void`. But be careful — `void` means "no value", while the function may still **modify external data** through pointers (covered in the next article).

### 3. No Arguments, With Return

```c
int getNumber() {
    return 42;
}

int main() {
    int number = getNumber();
    printf("The number is: %d\n", number);
    return 0;
}
```

### 4. With Arguments and Return

```c
#include <stdio.h>

int add(int a, int b) {
    int sum = a + b;
    return sum;
}

int main() {
    int num1 = 10, num2 = 20;
    int result = add(num1, num2);
    printf("The sum of %d and %d is %d\n", num1, num2, result);
    return 0;
}
```

## Function Categories

### 1. Library Functions

Provided by the C standard libraries, like `printf` and `scanf`:

```c
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
```

### 2. User-Defined Functions

Written by the programmer to perform specific operations:

```c
int add(int a, int b) {
    return a + b;
}
```

### 3. Inline Functions

Declared with `inline` to improve performance when called frequently:

```c
inline int square(int x) {
    return x * x;
}

int main() {
    int result = square(5);
    printf("The square of 5 is %d\n", result);
    return 0;
}
```

> **An important point for RE**: `inline` **is not a guarantee** — it is a request the compiler may ignore. And with optimization (`-O2`) the compiler may inline functions even without `inline`. **The result in the disassembly**: inlined functions vanish — no `call`, no stack frame, just their code copied into the caller. This explains why IDA's function list may not match what you wrote in the source.

## The Functions That Matter Most in Reverse Engineering

### 1. Memory Functions

They control allocation and management (`malloc`, `free`, `realloc`, `calloc`) — heavily used, and they reveal weak points:

| Function | Purpose |
|---|---|
| `malloc` | allocate heap memory |
| `calloc` | allocate + zero |
| `realloc` | reallocate with a new size |
| `free` | release memory |

> We will expand on them when covering memory management in later articles.

### 2. Input/Output Functions

They help you understand how data is read and written (`printf`, `scanf`, `fread`, `fwrite`, `gets`...) — analyzing them reveals input-handling weaknesses.

> Note: `gets` is considered one of the most dangerous functions ever — it reads with absolutely no bound on the input — and it has been removed from the modern standard.

### 3. String Functions

Common targets for buffer overflows (`strcpy`, `strcat`, `sprintf`...) — covered in detail in the arrays and strings article.

## What Is Buffer Overflow?

The vulnerability that occurs when incoming data exceeds a buffer's capacity: the excess spills into adjacent memory and corrupts it.

> **Analogy**: you have a juice bottle that holds exactly one liter (the buffer). Pour in more than a liter and the juice spills onto the stove (the adjacent memory), ruining it. The juice is the data; the stove is whatever sits after the buffer in memory.

## How Do Functions Appear in Assembly? (The Most Important Part)

This section is the heart of the article — because functions are the **direct bridge** from C to the disassembly.

### A. Function Structure: Prologue → Body → Epilogue

Every function becomes three blocks in assembly:

```asm
; PROLOGUE — building the stack frame
push ebp              ; save the caller's frame
mov  ebp, esp         ; ebp = top of stack (a fixed reference)
sub  esp, 16          ; reserve space for locals

; BODY — the actual code
mov  eax, [ebp+8]     ; read the first argument
add  eax, [ebp+12]    ; add the second argument
mov  [ebp-4], eax     ; store in a local variable

; EPILOGUE — tear down the frame and return
mov  esp, ebp         ; restore the stack
pop  ebp              ; restore the caller's frame
ret                   ; return to the caller
```

You will see this shape in **every** function when analyzing any binary — train your eye on it.

### B. Calling Conventions — The Difference Between Decoding and Understanding

When `add(5, 3)` is called, how are the values passed and who cleans up the stack? The conventions decide:

| Convention | Argument passing | Stack cleanup | Used by |
|---|---|---|---|
| `cdecl` | on the stack | the caller | C on Windows x86 |
| `stdcall` | on the stack | the callee | Win32 API |
| `fastcall` | first two args in ECX/EDX, rest on stack | the callee | Windows x86 |
| `x64` | RCX/RDX/R8/R9, then stack | the callee | Windows/Linux 64 |

**Why does this matter?** During analysis you will see two patterns:
- After a cdecl call you see `add esp, X` — the **caller** cleans the arguments.
- After a stdcall call you do not — the callee cleans up itself.

Understanding calling conventions is what makes disassembly readable instead of a random stream of instructions.

### C. Stack Frame Map

When a function is called, its frame is laid out at fixed offsets: locals sit at negative offsets from EBP, the return address sits at `[ebp+4]`, and arguments begin at `[ebp+8]` and upward.

> We will draw this map in full, tracing ESP instruction by instruction, in the **Virtual Memory (Stack vs Heap)** article — it is the same reference you will return to in every exploitation article later, so do not worry if it is not fully clear yet.

## Hands-on Exercise

1. Write an `add` function, then run `gcc -S -O0` — inspect the prologue/epilogue and note `[ebp+8]` and `[ebp+12]` holding the two arguments, with the return value placed in `eax`.
2. Try `-O2` — watch small functions disappear (inlining).
3. Call a Win32 function (such as `MessageBoxA`) and observe that the stack is not cleaned after the call (stdcall) — compare it with an ordinary C function (cdecl).



