---
title: "Bit Fields, Unions, and Enums"
description: "Bit fields, unions, and enums — three daily tools for reading binaries, each with clear patterns in assembly."
date: 2026-08-13T10:00:00+03:00
slug: "bit-fields-unions-enums"
weight: 18
hex: "0x17"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "bit-field", "union", "enum", "assembly"]
translationKey: "bit-fields-unions-enums"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we cover **Bit Fields**, **Unions**, and **Enums**.

## 1. Bit Fields

We established in the previous article that a struct distributes **bytes** — but what if we want to split a **single byte into bits**? That is the bit field: we specify the number of bits per member.

```c
typedef struct {
    u8 a : 4;   // 4 bits
    u8 b : 2;   // 2 bits
    u8 c : 1;   // 1 bit
} student;      // 7 bits total ← inside one byte
```

```text
+---+---+---+---+---+---+---+---+
| a (4 bits) | b (2) | c | (free) |
+---+---+---+---+---+---+---+---+
```

> **Note**: the compiler packs bit fields into the byte, but does not guarantee identical behavior across all compilers (implementation-defined). During analysis, reading bit fields in assembly involves `and`/`shift` operations.

## 2. Unions

A union means all members **share the same memory address** — every member starts at the same location, so it resembles a struct in shape but differs completely in behavior.

```text
high addresses
+-----------------+
|                 |
+-----------------+
|   u32 z         |
|   + u16 y       |
|   +   u8 x      |  ← all start at the same address
+-----------------+
low addresses
```

### Overlap Behavior

- Changing the `u8` changes the `u16` and `u32` (they all overlap).
- Changing the `u16` does **not necessarily** change the `u8` (it may lie outside it), but it **does change** the `u32`.

### Syntax

```c
union unionName {
    member1;
    member2;
};
```

### Union Size

```c
typedef unsigned char u8;
typedef unsigned short u16;
typedef unsigned int u32;

union myUnion {
    u8 x;
    u16 y;
    u32 z;
};

printf("the size of the myUnion is: %zu bytes\n", sizeof(myUnion));
```

**The output: 4 bytes** — a union takes **the size of its largest member** (not their sum).

> For comparison: as a struct it would be 7 bytes (packed) or 8 bytes (with padding). The union settles for the largest member because everyone shares the address.

### A Practical Example: A Register

```c
typedef union {
    struct {
        u8 B0 : 1;   // bit zero
        u8 B1 : 1;   // bit one
    } Bits;          // a struct of bit fields = 1 byte
    u8 Byte;         // the whole byte
} Register;
```

**Size = 1 byte** — both the struct (1 byte) and Byte (1 byte).

### Access

**1. Accessing the whole byte:**

```c
Register x;
x.Byte = 30;   // set the full value
```

**2. Accessing bit by bit:**

```c
x.Bits.B0 = 1; // set only bit zero
```

> Changing `B0` changes `Byte`'s value — they live at the same address.

## 3. Enums

An enum gives **names to numeric values** — making code more expressive. Instead of a reader seeing `7`, they see `apple`.

```c
enum fruit {
    mango,  // 0 (by default)
    apple   // 1
};
```

Values can be set manually:

```c
enum fruit {
    mango = 2,
    apple = 7
};
```

### A Graduated Example

```c
enum weekdays {
    sunday = 1,  // tell the compiler to start at 1
    monday,      // 2
    tuesday      // 3
};

enum weekdays w;
w = monday;
printf("the value of w is: %d\n", w); // 2
```

> **RE note**: in C an enum **is just an integer** (unlike some languages) — it appears in assembly as a constant (immediate).

## The RE Connection — Why This Article Is an Analytical Treasure

### 1. Bit Fields in Assembly = and + shift

Reading any bit field becomes a clear pattern:

```asm
mov al, [ebp-4]   ; read the whole byte
and al, 0x0F      ; extract the bits (e.g. B0)
shr al, 2         ; shift into position
```

> During analysis, consecutive `and` + `shr` **reveals bit fields** — like TCP/IP flags, PE header fields, and register flags.

### 2. Union = One Address, Different Sizes

In the disassembly, a union is **one address interpreted at different widths**:

```asm
mov eax, [esi]   ; reads a u32
mov al, [esi]    ; reads a u8 — from the same address!
```

> This matters when decoding network protocols and packet headers — the same bytes read as a flag or as a full value.

### 3. Enum = Embedded Constants (Immediates)

You will see `cmp eax, 7` instead of `cmp eax, apple` — and analysts rename these values with enum names in IDA/Ghidra to make the code readable. Values appear as immediates in cmp/switch.

### 4. The Register Example Is Real in RE

In practical analysis, registers and hardware layouts are read through unions like this one (byte/bit views of a register) — you will see the pattern in driver code and hardware initialization.

## Summary

- **Bit field**: splitting a byte into bits — appears in asm as `and`/`shift`.
- **Union**: members sharing one address — appears as different interpretations of the same location.
- **Enum**: names for numeric values — appears as embedded constants in cmp/switch.

All three are daily tools for reading binaries — master them and your reading of any code you analyze accelerates.











