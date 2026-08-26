---
title: "Data Modifiers"
description: "Sign, size, and storage modifiers (signed/short/static), and two's complement representation of negative numbers."
date: 2026-08-13T10:00:00+03:00
slug: "data-modifiers"
weight: 16
hex: "0x15"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "types", "numbers", "basics"]
translationKey: "data-modifiers"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we cover **data modifiers** — the keywords that change the properties and shape of data types:

- **Sign**: `signed`, `unsigned`.
- **Size**: `short`, `long`.
- **Storage**: `auto`, `register`, `extern`, `static`.

### General Notes

- You cannot combine more than one modifier of the same kind (`long short` together is invalid).
- Some modifiers are invalid in certain positions — like `unsigned float`, because floats are **always signed**.

## 1. Sign Modifiers

How is a signed variable represented inside a data type? Three methods evolved over time:

### (1) Signed Magnitude

The first 7 bits hold the value, and the last (highest) bit holds the sign — `0` positive, `1` negative:

```text
 5: 00000101
-5: 10000101
```

**Fatal flaws:**
- Zero has two representations: `+0` and `-0`.
- Arithmetic breaks: the ALU only knows **addition**, so what happens when adding `+1` to `-1`?

```text
+1: 00000001
-1: 10000001
    10000010 = -2   ← wrong! It should be 0
```

The temporary workaround was conditional logic: if `P == Q` the result is 0, if `P > Q` subtract, and so on — **unacceptable complexity**.

### (2) One's Complement

Flip all bits (as if applying NOT):

```text
+1: 00000001
-1: 11111110
```

**Same problems**: `+0` and `-0` both exist, and adding `+1` to `-1` yields `-0` (11111111).

### (3) Two's Complement — The Adopted Method

If the first bit is `1` (a negative number), flip the remaining bits and add 1:

```text
 5: 00000101
-5: 11111011
```

**Why it is best?**
- **Exactly one zero** — no `+0` and `-0`.
- The ALU **only adds**: adding `5` to `-5` produces a discarded overflow, leaving 8 bits = 0.
- Subtraction = addition of the complement — which is why **every modern processor adopted it** (x86, ARM...).

```text
  00000101  (5)
+ 11111011  (-5)
 ----------
  100000000  → keep 8 bits = 00000000 = 0
```

### Key Rules

- `unsigned` applies to all integer types: `char`, `short`, `int`, `long`, `long long`.
- `float` and `double` are **always signed** — there is no unsigned version.

## 2. Size Modifiers

| Type | Common size |
|---|---|
| `char` | 1 byte |
| `short` | 2 bytes |
| `int` | 4 bytes |
| `long` | 4 bytes (Windows) / 8 bytes (Linux x64) |
| `long long` | 8 bytes |
| `float` | 4 bytes |
| `double` | 8 bytes |
| `long double` | 16 bytes (GCC/Linux) / 8 bytes (MSVC) |

- `long` and `short` imply `int` by default (i.e. `long int`, `short int`).
- **Important**: the sizes of `long` and `long double` depend on platform and compiler — as noted in the data types article.

### The Compiler Divergence Problem — The Fix: typedef

What if one compiler says `char` = 1 byte and another says 2? The result is portability errors. The fix: **standardized types** via `typedef` with a **naming convention** (type + bit width):

```c
// stdtypes.h
typedef unsigned char  u8;   // unsigned 8-bit  = 1 byte
typedef unsigned short u16;  // unsigned 16-bit = 2 bytes
typedef unsigned int   u32;  // unsigned 32-bit = 4 bytes
typedef unsigned long long u64; // unsigned 64-bit = 8 bytes
```

Usage:

```c
u16 varName; // an unsigned variable, 16 bits wide
```

> **RE note**: these patterns underlie the Windows SDK types: `BYTE` (1), `WORD` (2), `DWORD` (4), `QWORD` (8) — you will see them in every PE and API analysis.

## 3. Storage Modifiers

While code runs, data may move between register, cache, and RAM — depending on size and declaration. At the low level: a variable moves from RAM into a register for processing and back.

### Variable Kinds

| Modifier | Storage | Scope | Lifetime |
|---|---|---|---|
| `auto` | RAM | local/global | with the function / whole program |
| `register` | Register (sometimes RAM) | local only | with the function |
| `static` | RAM (`.data`/`.bss`) | local/global | whole program |
| `extern` | no storage of its own — mirrors a definition in another file | local/global | whole program |

- **auto**: just like writing plain `int` — the default for locals.
- **register**: a **hint** to the compiler to keep the variable in a register. Modern compilers **usually ignore it** — they decide themselves, and when variables exceed available registers they spill to RAM.
- **static**: stored in `.data` (initialized) or `.bss` (uninitialized) — covered in detail in the previous article.
- **extern**: allocates no memory — it references a definition in another file, resolved during **linking**.

## The RE Connection

1. **signed vs unsigned in disassembly**: the value `0xFFFFFFF0` reads as `-16` signed or `4294967280` unsigned — **and this completely changes how you read the code**. The distinction surfaces in assembly via `movsx` (sign extension) vs `movzx` (zero extension) when widening `char`/`short`.

2. **Two's complement is fundamental**: knowing that `0xFFFFFFFF` = `-1` signed explains why you see `inc`/`dec` or descending address arithmetic in the disassembly. An analyst who does not know the representation misreads signed values.

3. **typedef types in binaries**: RE tools display types like `uint32_t` or `DWORD` — mastering the name-to-size mapping is a daily reference for analysts.

4. **register and allocation**: the optimization level (`-O0` vs `-O2`) determines where variables live (stack vs registers) — which is why the disassembly changes radically with optimization level, and why you see variables in different registers depending on context.







