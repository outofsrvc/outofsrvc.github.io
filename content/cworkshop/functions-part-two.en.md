---
title: "Functions — Part Two"
description: "Variable scope, header files, static, and recursion — with a complete step-by-step trace of the stack frame."
date: 2026-08-13T10:00:00+03:00
slug: "functions-part-two"
weight: 15
hex: "0x14"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "assembly", "functions", "recursion", "static"]
translationKey: "functions-part-two"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

Having laid the cornerstone of C, memory division, pointers, and the assembly introduction — we continue toward a complete body of knowledge. This article treats functions at an **advanced** level, with K&R as a primary reference, tracing registers in the assembly to see how the code moves at the low level.

**Article index:**
- 1.1 Variable Scope
- 1.2 Header Files
- 1.3 Static Variables and Functions
- 1.4 Recursion

## Function Essentials

| Component | Description |
|---|---|
| **Signature** | the declaration: `returnType funcName(Parameters) {}` |
| **Body** | what the function does |
| **Parameters** | input values |
| **Return value** | the main output |

## A Function's Execution Model

```c
int add(int a, int b) {
    int result = a + b;
    return result;
}
```

### The Assembly (32-bit)

We use `-m32` to simplify the explanation (the classic EBP pattern seen in millions of legacy programs):

```bash
gcc -S -masm=intel -m32 -fno-asynchronous-unwind-tables -fno-pie -O0 file.c -o file.s
```

> On Arch Linux: `sudo pacman -S gcc-multilib lib32-gcc-libs lib32-glibc`
> On Kali/Debian: `sudo apt install gcc-multilib libc6-dev-i386 libc6:i386`

```asm
add:
    push    ebp
    mov     ebp, esp
    sub     esp, 16
    mov     edx, DWORD PTR [ebp+8]   ; edx = a = 5 (first argument)
    mov     eax, DWORD PTR [ebp+12]  ; eax = b = 3 (second argument)
    add     eax, edx                 ; eax = 3 + 5 = 8
    mov     DWORD PTR [ebp-4], eax   ; result = 8
    mov     eax, DWORD PTR [ebp-4]   ; eax = result (return value)
    leave                            ; mov esp, ebp + pop ebp
    ret
main:
    push    ebp
    mov     ebp, esp
    push    3                        ; passes b = 3 first (right-to-left)
    push    5                        ; then a = 5
    call    add
    add     esp, 8                   ; cleaning the arguments — the caller (cdecl)
    mov     eax, 0
    leave
    ret
```

### Step by Step

**1. Creating add's stack frame:**

```asm
add:
    push ebp      ; save the current EBP (main's)
    mov  ebp, esp ; establish a new frame — the first two instructions in most functions
```

**2. Reserving space for locals:**

```asm
sub esp, 16      ; 16 bytes (result is 4 bytes + the rest alignment/padding)
```

**3. Reading the inputs and computing:**

```asm
mov edx, DWORD PTR [ebp+8]   ; edx = 5 (first argument a)
mov eax, DWORD PTR [ebp+12]  ; eax = 3 (second argument b)
add eax, edx                 ; eax = 3 + 5 = 8
```

**4. Storing and returning the result:**

```asm
mov DWORD PTR [ebp-4], eax   ; store the result in the local
mov eax, DWORD PTR [ebp-4]   ; load the result into eax (return value)
leave                        ; mov esp, ebp + pop ebp
ret                          ; pop the return address back to main
```

### Tracing ESP (the Stack Map)

**1. `main` starts (after `mov ebp, esp`):**

```text
[ret address]   : return address into the system
[saved ebp]     : the current ebp/esp
```

**2. After `push 3` + `push 5`:**

```text
[ret address]
[saved ebp]
3
5            ← esp here
```

**3. After `call add` (which places the return address):**

```text
[ret address]
[saved ebp]
3
5
[ret address]   : return address into main
```

**4. Inside add after `mov ebp, esp`:**

```text
[ret address]
[saved ebp]
3
5
[ret address]
[saved new ebp]  : ebp/esp belonging to add
```

**5. After `sub esp, 16`:**

```text
[ret address]
[saved ebp]
3               : [ebp + 12]  → b
5               : [ebp + 8]   → a
[ret address]   : [ebp + 4]
[saved new ebp] : [ebp]
[result]        : [ebp - 4]
[padding]       : [esp]
```

**6. After `leave`** — esp returns to saved ebp's position, then it is restored.
**7. After `ret`** — jumps to main's return address.
**8. After `add esp, 8`** — cleans arguments 3 and 5.

> **Note**: `add esp, 8` after the call means this is **cdecl** (the caller cleans). If it were stdcall you would see `ret 8` inside the function instead.

## 1.1 Variable Scope

Scope determines where a variable can be accessed in the code.

### 1. Block Scope (Local)

**Where**: inside a `{}` block (function, loop, condition...).
**Access**: visible only inside the block, invisible outside it.

```c
void example() {
    int x = 10;          // local to the function
    if (x > 5) {
        int y = 20;      // local to the if
        printf("%d", y); // prints
    }
    printf("%d", y);     // error — y is not visible here
}
```

### 2. File Scope (Global)

**Where**: outside any function (at file level).
**Access**: reachable from anywhere in the file.

```c
int global = 50;

void func1() { global++; }      // allowed
void func2() { printf("%d", global); } // prints 51
```

### 3. Parameter Scope

**Where**: inside the function's `()`.

```c
int sum(int a, int b) {  // a, b live inside this function only
    return a + b;
}
```

### 4. Shadowing

Declaring a local with the same name as a global hides the global and uses the local copy.

```c
int x = 100; // a global variable

void demo() {
    int x = 200;     // shadows the global
    printf("%d", x); // prints 200
}
```

### Important Notes

1. Variables are declared at the start of their `{}` block.
2. Global variables are declared outside functions.
3. Scope runs from the point of declaration to the end of the block (local) or the end of the file (global).
4. Nested blocks can reach outer-block variables unless shadowing occurs.

## 1.2 Header Files

`.h` files (the include files) hold declarations shared across several `.c` source files — avoiding repeated declarations and separating the interface from the implementation.

### What Header Files Contain

1. **Function declarations** (no body):

```c
int add(int a, int b);
```

2. **External variables** (extern):

```c
extern int counter;
```

3. **Macros**:

```c
#define MAX_SIZE 100
```

4. **Struct and union definitions** (upcoming).
5. **Type definitions** (typedef — upcoming).
6. **System library includes**:

```c
#include <stdio.h>
```

### A Practical Example (Separating Interface from Implementation)

**calc.h** — the header:

```c
#ifndef CALC_H
#define CALC_H

// function declaration
int add(int a, int b);

// external variable declaration
extern int result;

#endif
```

**main.c** — usage:

```c
#include "calc.h"

int result = 0; // the actual definition (exists in exactly one file)

int main() {
    result = add(5, 3);
    return 0;
}
```

**math.c** — implementation:

```c
#include "calc.h"

int add(int a, int b) {
    return a + b;
}
```

> **Declaration vs Definition**: `extern` merely declares "a variable with this name exists somewhere" — the definition is what actually allocates memory (`int result = 0;`) and must appear in exactly one file. This distinction matters in RE when telling declared symbols from defined ones.

### Include Guards

They prevent repeated inclusion when a file is included more than once (preventing conflict errors):

```c
#ifndef MY_HEADER_H  // if this symbol is not defined
#define MY_HEADER_H  // define it now
// file contents
#endif
```

### Important Warnings

- **Never** put definitions in headers (like `int x;`).
- Use `extern` for shared variables.
- **Never** put function bodies in the header — they belong in the implementation file.

## 1.3 Static Variables and Functions

`static` is used in three contexts:

### 1. A Static Variable at File Scope

Visibility confined to the current file — inaccessible from other files even with `extern`:

```c
// in file1.c
static int internal_var; // invisible outside the file

void func() {
    internal_var = 42;
}
```

### 2. A Static Variable Inside a Function

Keeps its value **between calls** — initialized only once (first call) and persists for the whole program run:

```c
void counter() {
    static int count = 0; // initialization happens exactly once
    count++;
    printf("Count: %d\n", count);
}
```

First call → `count = 1`, second → `count = 2`.

### 3. A Static Function

Visible only within the file where it was defined — it cannot be called from other files:

```c
static void helper() {
    // local to this file only
}
```

### Automatic Initialization and Storage

Uninitialized static variables are automatically zeroed:

```c
static int x;  // x = 0
static int *p; // p = NULL
```

> **Storage location**:
> - an **initialized** static → `.data` segment.
> - an **uninitialized** static (zeroed) → `.bss` segment.
>
> This matches what we covered in memory layout — and it marks a fixed address in the binary rather than an offset from EBP.

### Comparison Table

| | **static** | **extern** |
|---|---|---|
| Visibility | current file only | all files |
| Storage | `.data`/`.bss` | `.data`/`.bss` |
| Initialization | auto-zeroed if unset | depends on the definition |
| Purpose | hiding details | sharing across files |

## 1.4 Recursion

A technique where a function calls **itself** to solve a problem — breaking the main problem into smaller sub-problems of the same kind until they become trivially solvable.

It rests on two principles:
1. **Base Case**: the condition stopping recursion — the simplest case solved directly (`0! = 1`).
2. **Recursive Step**: splitting the problem into a simple part + a smaller call to the same function.

### Example: Factorial

```c
#include <stdio.h>

int factorial(int n) {
    if (n == 0)              // base case
        return 1;
    else                     // recursive step
        return n * factorial(n-1); // recursion
}

int main() {
    printf("%d! = %d\n", 3, factorial(3)); // prints 6
    return 0;
}
```

### Step by Step

```text
1. factorial(3): 3 != 0 → 3 * factorial(2)
2. factorial(2): 2 != 0 → 2 * factorial(1)
3. factorial(1): 1 != 0 → 1 * factorial(0)
4. factorial(0): 0 == 0 → 1

Unwinding: 1*1 = 1 → 1*2 = 2 → 2*3 = 6
```

## The RE Connection — Why This Article Is Essential

1. **`[ebp+4]` = Return Address = the exploitation target**: the stack map we drew is exactly what **buffer overflow** exploits — an overflowing input writes over the locals ← saved EBP ← **the return address**, redirecting execution to shellcode or a ROP chain. This article builds the understanding the exploitation content will explain.

2. **Recursion in assembly = stacked frames**: every self-call adds a new frame (call/ret + prologue). A missing base case = **Stack Overflow** (stack exhaustion) — also a DoS attack technique. In a binary, recursion appears as repeated calls to the same address.

3. **Static variables = fixed addresses**: in disassembly they appear as **absolute addresses in `.data`/`.bss`** rather than offsets from EBP — a quick visual tell in IDA that reveals globals instantly.

4. **Shadowing leaves no trace in asm**: it resolves at compile time — the compiler may reuse the same slot. You will not see it in compiled code.

5. **Header files and their binary footprint**: declaration vs definition surfaces in symbols — and separating implementation from interface means shared functions appear as imports/exports in the PE.

---

God willing these concepts are clear, and they will stay with us in every program we write. Apologies if the articles run long — but this is knowledge's due right.












