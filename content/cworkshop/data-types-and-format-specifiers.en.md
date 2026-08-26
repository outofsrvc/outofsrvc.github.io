---
title: "Data Types and Format Specifiers"
description: "Data types and their real sizes on x86/x64, plus printf/scanf format specifiers — mapped to operand sizes in assembly."
date: 2026-08-13T10:00:00+03:00
slug: "data-types-and-format-specifiers"
weight: 4
hex: "0x03"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "architecture"]
translationKey: "data-types-and-format-specifiers"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will look at data types and the format specifiers associated with them.

> **Note**: sometimes memorizing the English terminology is a must, because most of your work with programming languages will be in English — and Arabic translations vary from one person to another.

## What Are Data Types?

They are the form in which data (such as variables) is stored so it can be kept and processed. The most important types:

### 1. Integers

- Used to store whole numbers (1, 55, -72 — no decimal points).
- Keyword: `int`.
- Size: **4 bytes on both x86 and x64** (see the common-mistakes note below).
- Numbers can be made positive-only using `unsigned`.

### 2. Floating-point

- Used to store numbers with decimal fractions (1.2, 3.7).
- `float`: 4 bytes.
- `double`: 8 bytes.

### 3. Characters

- Stores a single character (a letter, digit, or any symbol).
- Keyword: `char`.
- Size: 1 byte.

### 4. Void (no value)

- Used with functions or pointers that do not return a value.
- Keyword: `void`.
- **Important for RE**: `void*` is a generic pointer (of unspecified type) — we will return to it in the final basics articles because it is one of the most important things in reverse engineering.

## Data Size Table (on Windows x64)

| Type | Size (bytes) | Approximate range |
|---|---|---|
| `char` | 1 | -128 to 127 |
| `unsigned char` | 1 | 0 to 255 |
| `short` | 2 | -32,768 to 32,767 |
| `int` | 4 | -2.1 billion to 2.1 billion |
| `unsigned int` | 4 | 0 to 4.3 billion |
| `long` | 4 (Windows) / 8 (Linux x64) | depends on the system |
| `long long` | 8 | huge numbers |
| `float` | 4 | decimal fraction |
| `double` | 8 | decimal fraction, higher precision |
| `pointer` | 4 (x86) / 8 (x64) | a memory address |

### A Common Mistake to Avoid

> **The size of `int` does not depend on the architecture (x86/x64)** — it depends on the compiler/ABI. `int` is 4 bytes on both x86 and x64, on Windows and Linux alike. The real difference between architectures shows up in pointer sizes (4 bytes on x86, 8 bytes on x64) and in `long` on some systems.

### Verify It Yourself

```c
#include <stdio.h>

int main(void){
    printf("char: %zu bytes\n", sizeof(char));
    printf("int:  %zu bytes\n", sizeof(int));
    printf("long: %zu bytes\n", sizeof(long));
    printf("ptr:  %zu bytes\n", sizeof(void*));
    return 0;
}
```

> Note that `sizeof(void*)` will print 8 on a 64-bit system and 4 on a 32-bit system — this is the practical way to determine the system's architecture.

## What Are Format Specifiers For?

They are used in functions like `printf` and `scanf` to format data input or output:

- `printf`: the printing function (output).
- `scanf`: the reading function (input).

### 1. Integer Specifiers

| Specifier | Description |
|---|---|
| `%d` | signed integer (decimal) |
| `%u` | unsigned integer |
| `%x` / `%X` | hexadecimal — lowercase/uppercase |
| `%o` | octal |
| `%ld` / `%lld` | signed `long` / `long long` |
| `%zu` | `size_t` (used with `sizeof`) |

### 2. Floating-point Specifiers

| Specifier | Description |
|---|---|
| `%f` | floating-point number (float/double) |
| `%.1f` | floating point with one digit after the decimal point |
| `%e` | scientific notation |
| `%g` | automatic representation |

### 3. Character and String Specifiers

| Specifier | Description |
|---|---|
| `%c` | single character |
| `%s` | string |
| `%c` with `%d` | printing a character as its ASCII number |

### 4. Other Important Specifiers

| Specifier | Description |
|---|---|
| `%p` | a pointer (memory address) — **not `%x`** |
| `%%` | printing the `%` sign itself |

## Connecting This to Reverse Engineering

1. **`int` is always 4 bytes**: when you see an instruction in the disassembly reading from `[ebp-4]` or `[rsp+8]` with a size of 4 bytes, it is most likely an `int` variable. This mapping between C's sizes and opcodes is the first skill of reading disassembly.

2. **`%d` vs `%u`**: the former prints the value as signed and the latter as unsigned. When decoding data or analyzing hex values, whether a value is interpreted as signed or unsigned matters enormously — the difference can flip your entire analysis.

3. **`char` vs `int`**: dealing with 1 byte versus 4 bytes explains why you see `movzx` (zero extension) and `movsx` (sign extension) instructions in assembly when moving a `char` into an `int`. We will cover them in detail in the assembly series.

## Closing Note

Do not worry if memorizing format specifiers feels hard — with frequent use they become automatic routine.

> **Note**: we did not cover the `%p` specifier in detail here because we will get to pointers in the final articles of the basics — and they are among the most important topics for reverse engineering. May Allah grant you success.
