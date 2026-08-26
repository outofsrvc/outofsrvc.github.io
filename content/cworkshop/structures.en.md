---
title: "Structures"
description: "Structs, typedef, and the arrow operator, padding and alignment — and how a struct becomes an offset map during analysis."
date: 2026-08-13T10:00:00+03:00
slug: "structures"
weight: 17
hex: "0x16"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "struct", "memory", "assembly"]
translationKey: "structures"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we cover the **Structure** — the first user-defined data type.

## Struct Basics

We know C ships with standard data types: `int`, `char`, `float`, `double`...

What if we want to represent a **real-world object** — like a person with a **name** and an **age**? We need to mix a string (array of characters) with an int inside one object.

That is where the **Structure** comes in: it resembles an array, but it **groups variables of multiple types** (whereas arrays hold one type). Since the developer designs it, it counts as a **User Defined Data Type**.

### Declaring a Structure

```c
struct structName {
    dataType member1;
    dataType member2;
    // ...
    dataType memberN;
};
```

### Declaration and Initialization

```c
struct human {
    int age;
    char name[10];
};

struct human h1, h2;              // declaring the variables
h1.age = 33;                      // filling members later, one by one
struct human h3 = {33, "mj3s"};   // initialization at declaration (member order)
```

## Accessing Members

There are two main ways to access struct members:

### 1. The Dot Operator (`.`)

```c
struct human h1;
h1.age = 36;      // access through the object directly
```

### 2. The Arrow Operator (`->`)

Used with a **pointer** to a struct:

```c
struct human *ptr = &h1;
ptr->age = 36;      // access through the pointer
```

> **The arrow's equivalent**: `(*ptr).age` means the same as `ptr->age` — the arrow is shorthand for dereference + dot. The two styles are equivalent, and the arrow is the more common.

## Arrays of Structures

```c
struct human hmn[10]; // an array of 10 human objects
```

## Assignment on Structs

Arithmetic does not apply to structs — **with the exception of assignment** `=`:

```c
struct human h1 = {33, "mj3s"};
struct human h2;
h2 = h1; // copies every member — valid
```

## typedef — An Alias

**typedef**: gives an existing data type an alias name.

```c
typedef <existingName> <aliasName>;
```

Example:

```c
typedef unsigned int u32;
u32 x; // a variable of type unsigned int
```

### Improving Struct Ergonomics with typedef

Instead of writing `struct student` every time:

```c
typedef struct student {
    int age;
    int light;
} stud;
```

Now we use the alias directly:

```c
stud s1, s2;       // instead of struct student s1, s2;
stud ahmed = {10, 30};

int main() {
    stud *ptr = &ahmed;
    printf("ahmed age is %d\n", ptr->age);
    return 0;
}
```

## Passing Structs to Functions

Two ways:

```c
void f1(struct human h) {   // by value — copies every member
    h.age = 99;             // does not affect the original
}

void f2(struct human *h) {  // by reference (the more efficient way)
    h->age = 99;            // affects the original
}
```

> **Why is reference passing more efficient?** Passing a large struct by value copies every member onto the stack — while the reference passes a single address.

## Structure Padding

### The Width and Depth Concept

Memory has **width** (how many bytes each access reads) and **depth** (the number of locations):
- A 32-bit processor: width = 4 bytes (each access fetches 4 bytes).
- A 64-bit processor: width = 8 bytes.

Memory space = width × depth. Example: 4-byte width × 1000 depth = **4000 bytes**.

### The Problem Without Padding

Suppose:

```c
struct student {
    char a;   // 1 byte
    short b;  // 2 bytes
    short c;  // 2 bytes
};
```

Without padding, the variable `b` (2 bytes) could straddle two memory locations — requiring **merge operations** that may reach 7 instructions in assembly just to read one value!

### The Fix: Padding

The compiler leaves **empty gaps** (padding) so variables align to suit the memory width — and get read in a single access:

- **Advantage**: much higher performance.
- **Disadvantage**: more memory used.

### How Ordering Affects Padding

**Member order changes the total size**:

```c
struct bad {
    char a;   // 1
    int  b;   // 4
    char c;   // 1
};  // sizeof = 12 (not 6!)
```

The raw space is 6 bytes, but alignment inflates it to **12 bytes** — wasting 6.

> **Golden rule**: order members from largest to smallest (int, short, char) to minimize padding and save memory.

### Disabling Padding: packed

To force the compiler not to add padding (zero alignment):

```c
struct base {
    // body
} __attribute__((packed));
```

> **When to use packed?** In file formats and protocols demanding strict layouts without gaps — stored files cannot tolerate padding.

## The RE Connection — Why Structs Are Fundamental

1. **A struct in assembly = offsets**: a struct turns into **fixed offset arithmetic** from its base address. The access `ptr->age` becomes:

```asm
mov eax, [edi+8]   ; reading member age at offset 8
mov edx, [edi+12]  ; reading member name at offset 12
```

During analysis, seeing repeated `[reg+offset]` patterns on the same register reveals a struct — an essential skill for reading any disassembly.

2. **Padding explains sizes**: seeing `sizeof(struct)` = 12 while the members sum to 6 — **the analyst needs the same calculation** to predict buffer sizes and where members begin. Misjudging sizes is a source of errors in heap and stack analysis.

3. **packed in RE**: packed structs appear in file formats and protocols (files tolerate no padding) — knowing them helps when analyzing file formats and decoding protocols.

4. **Members = adjacent variables in memory**: understanding member order in memory explains why an overflow of one member spills into the next ones (the basis of heap/stack struct corruption).

## Summary

The struct is **C's fundamental data-organizing unit** — and in RE it becomes a simple yet pivotal offset map. Master padding calculations and member ordering, and you will read any struct's layout in the disassembly instantly.









