---
title: "Pointers and Memory Management"
description: "Pointer types, address arithmetic, and the five memory segments — plus where pointers meet famous vulnerabilities."
date: 2026-08-13T10:00:00+03:00
slug: "pointers-and-memory"
weight: 10
hex: "0x09"
stage: "memory"
categories: [c-lang]
tags: ["c-lang", "pointers", "memory", "heap", "stack"]
translationKey: "pointers-and-memory"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will cover pointers and their types, along with an introduction to memory management and allocation. This is **the final article of the basics series**, God willing.

> **Note**: we will not expand into memory management as deeply as the topic deserves — we cover the core concepts that help us understand pointers, because the subject warrants its own article.

Pointers are a powerful tool giving direct control over memory, which makes them of paramount importance when analyzing programs and understanding binary instructions.

## Memory Layout

Before explaining pointers, you need background on memory layout and allocation. In C, every variable is stored at a specific memory location with a unique address: locals live on the stack, globals in the data section — and a pointer gives you access to a variable's address, granting precise control over how the program uses memory.

### Memory Segments

| Segment | Purpose | Notes |
|---|---|---|
| **Text Segment** | holds the compiled executable instructions (machine code) | read-only |
| **Data Segment** | initialized globals/constants | `int x = 10;` |
| **BSS Segment** | uninitialized globals/constants | `int x;` ← defaults to 0 |
| **Stack** | local variables and function bookkeeping | temporary, LIFO, discarded when the function ends |
| **Heap** | dynamic allocation | manual, persists until `free` |

```text
+-----------------+  high addresses
|      Stack      |  grows downward (temporary allocation)
|        |        |
|        v        |
|        ^        |
|      Heap       |  grows upward (manual allocation)
+-----------------+
|      BSS        |  uninitialized
|     Data        |  initialized
|     Text        |  read-only
+-----------------+  low addresses
```

- **Stack**: LIFO (last in, first out). Operations: `push` (add) and `pop` (remove). Temporary — its space is reclaimed as soon as the function ends.
- **Heap**: dynamic allocation via `malloc`, `calloc`, `realloc` — remains reserved until `free`.

**Management-wise**: the stack, data, and BSS are managed automatically by the system, while the heap requires manual management by the programmer.

**Performance-wise**: the stack is faster than the heap because of how simple its management is, but its size is limited compared to it.

## Memory Allocation

Reserving space in the computer's memory to store data while the program runs. In C there are three kinds:

| Kind | When | Where | Lifetime |
|---|---|---|---|
| **Static** | compile time | Data/BSS | the whole program run |
| **Automatic** | when the function is called | Stack | until the function returns |
| **Dynamic** | during execution | Heap | until `free` |

```c
static int x = 10;      // static — data segment
int y;                  // automatic — stack
char *p = malloc(100);  // dynamic — heap
free(p);                // must be freed manually
```

> **The programmer's responsibility**: C has no garbage collection like Java — the programmer must free dynamic memory manually to avoid memory leaks.

## Pointers

Pointers are variables that store **memory addresses** of other variables. If you have an `int` variable, you can create a pointer holding its address and then read or modify its value indirectly — which is what makes them such a powerful tool for managing memory.

```c
data_type *pointer_name;
```

### How to Use Pointers

**1. Declare the pointer and bind it to an address:**

```c
int a = 10;     // an ordinary variable
int *p = &a;    // declare the pointer and bind it to the variable's address
// &a  : yields the address of a
// p   : holds the address of a
// *p  : reaches the value stored at that address
```

**2. Dereferencing:**

```c
printf("Value of a: %d\n", *p); // *p gives the value of a: 10
```

**3. Modifying the value through the pointer:**

```c
*p = 20;   // change a's value through the pointer
printf("New value of a: %d\n", a); // output: 20
```

### A Comprehensive Example

```c
#include <stdio.h>

int main() {
    int x = 10;
    int *ptr = &x;
    printf("x value: %d\n", x);          // 10
    printf("x address: %p\n", &x);       // the memory address
    printf("ptr value: %p\n", ptr);      // same as x's address
    printf("value via ptr: %d\n", *ptr); // 10
    return 0;
}
```

## Pointer Types

### 1. Basic Pointers

```c
int x = 5;
int *ptr = &x;
```

### 2. Pointer to Pointer

```c
int x = 10;
int *ptr = &x;
int **ptr2 = &ptr; // pointer to the pointer ptr
printf("%d", **ptr2); // 10
```

### 3. Array Pointers

The array name decays into a pointer to its first element in expressions. **`a[i]` is equivalent to `*(a + i)`**:

```c
int arr[] = {1, 2, 3};
int *ptr = arr; // points at the first element
```

### 4. Function Pointers

They point at the starting address of a function, enabling dynamic invocation:

```c
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int main() {
    int (*func_ptr)(int, int) = add;
    int result = func_ptr(3, 4);
    printf("Result: %d\n", result);  // 7
    return 0;
}
```

### 5. Void Pointers

Can point at data of any type:

```c
void *ptr;
int a = 10;
ptr = &a; // now points at an integer
```

> **Note**: before using a `void *` it must be cast to a concrete type — otherwise you cannot know the size of the data it points at.

### 6. Constant Pointers — An Important Distinction

Three similar-looking kinds must be told apart precisely (because the assembly reveals which one was used):

```c
int *const ptr = &x;    // constant pointer: cannot be re-pointed, but x's value can change
const int *ptr = &x;    // constant value: the pointer itself can change, but not x's value
const int *const ptr;   // both constant
```

## Address Arithmetic

Adding to and subtracting from pointers moves by multiples of the **data type's size**:

```c
int arr[3] = {10, 20, 30};
int *ptr = arr;
printf("%d\n", *ptr);  // 10
ptr++;                 // moves 4 bytes forward (the size of int)
printf("%d\n", *ptr);  // 20
```

> Remember from the data types article: incrementing an int pointer moves 4 bytes; incrementing a char pointer moves 1 byte.

## Pointers with Functions (Passing by Reference)

Passing an address to a function lets it modify the original values — because ordinary arguments are passed **by value** (a copy):

```c
#include <stdio.h>

void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

int main() {
    int x = 5, y = 10;
    swap(&x, &y);   // pass the addresses
    printf("x = %d, y = %d\n", x, y); // x = 10, y = 5
    return 0;
}
```

## Pointers and Strings

```c
char *str = "Hello, World!";
printf("%s\n", str);        // the full text
printf("%c\n", *str);       // H — the first character
printf("%c\n", *(str + 1)); // e — the second character
```

> **An important point for RE**: string literals are stored in a **read-only** section (`.rodata`). Trying to modify them through the pointer **crashes the program** — which is why the compiler may warn you, and why that segment shows up read-only in binaries.

## Pointers in Security Vulnerabilities

### 1. Buffer Overflow

Allocating a bounded space and overflowing it with an unsafe pointer:

```c
void vulnerable_function(char *input) {
    char buffer[10];
    strcpy(buffer, input); // buffer size can be exceeded
}
```

In reverse engineering these vulnerabilities are analyzed either to exploit them or to understand how attacks work.

### 2. Use After Free (UAF)

Freeing a pointer and then using it later:

```c
char *ptr = malloc(10);
free(ptr);
strcpy(ptr, "data"); // using the pointer after freeing
```

### 3. Null Pointer Dereference

Dereferencing a null address crashes the program:

```c
int *ptr = NULL;
*ptr = 5; // crash
```

### 4. Double Free

Freeing the same pointer twice — can lead to allocator vulnerabilities:

```c
char *ptr = malloc(10);
free(ptr);
free(ptr); // freed twice
```

> UAF and Double Free are the foundation of **heap exploitation** (such as tcache poisoning) — a topic we will expand on in a future series.

## Pointers in Assembly — The Series' Crowning Section

This section ties everything learned in the series to the disassembly:

### A. A Pointer = an Addressing Mode

| C expression | Assembly | Meaning |
|---|---|---|
| `*ptr` (read) | `mov eax, [ebx]` | read the value at the address |
| `ptr[i]` | `mov eax, [ebx + ecx*4]` | pointer + offset (index × size) |
| `&x` | `lea eax, [ebp-4]` | compute the address without reading the value |
| `*ptr = val` (write) | `mov [ebx], eax` | write the value to the address |

- `mov eax, [address]` = dereference.
- `lea` = obtaining the address itself — **the most important tool** for pointers.

### B. Passing by Reference in Assembly

When `swap(&x, &y)` is called, the disassembly looks like:

```asm
lea eax, [ebp-8]   ; address of y
push eax
lea eax, [ebp-4]   ; address of x
push eax
call swap

; inside swap:
mov eax, [ebp+8]   ; read the address (not the value!)
mov ecx, [eax]     ; read the actual value from that address
```

Arguments passed by reference appear as **addresses on the stack** — note the difference: `[ebp+8]` gives you the address, then `[eax]` gives you the real value.

### C. A Function Pointer = a Code Address

A function pointer in the disassembly is a direct address of code, and it may appear in tables (jump tables, or vtables in C++). Tracing these addresses is part of malware analysis — especially in API-hiding techniques.

### D. The Final Takeaway

In reverse engineering, pointers are **the key to understanding data flow** between parts of a program — because they translate directly into addressing patterns in the assembly that can be recognized and linked to vulnerabilities.

## The Series' Closing Exercise

1. Write `int *ptr = &x;` and `printf` for each of `x`, `&x`, `ptr`, `*ptr` — then inspect the generated assembly and identify the `lea` and `mov [..]` patterns.
2. Write `swap` and inspect it with `gcc -S` — notice `[ebp+8]` and `[ebp+12]` hold the addresses, and modification happens via `[eax]`.
3. Try modifying a string literal (`str[0] = 'X'`) — observe the crash and explain it using your knowledge of memory sections (`.rodata` is read-only).

---

The C basics series ends here. God willing, there will be an advanced series with much more depth — and the next stage will focus on system-level matters, because C's real power lies in how it deals with the OS and its speed.







