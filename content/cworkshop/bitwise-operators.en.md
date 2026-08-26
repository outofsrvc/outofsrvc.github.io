---
title: "Bitwise Operators"
description: "Bitwise operators and increment/decrement — the shared language between C and assembly, including XOR decryption patterns."
date: 2026-08-13T10:00:00+03:00
slug: "bitwise-operators"
weight: 11
hex: "0x10"
stage: "memory"
categories: [c-lang]
tags: ["c-lang", "bitwise", "assembly", "low-level"]
translationKey: "bitwise-operators"
ShowToc: true
TocOpen: false
draft: false
---

Assalamu alaikum wa rahmatullahi wa barakatuh.

From this article on we deal with C at a genuinely **low level**.

> **Before starting**: you should be comfortable with the basics of C — you can review the previously published basics series.

In this article we cover bitwise operators plus increment/decrement and the ternary operator. We will not focus on simple arithmetic (addition, subtraction...) because it is self-evident — this series focuses on the more powerful constructs that serve us in reverse engineering.

## What Are Bitwise Operators, and Why Learn Them?

Operators used to manipulate individual bits in numbers. They are fundamental to low-level programming because they work directly with binary (0 and 1) — exactly as the processor does.

## 1. AND (`&`)

Compares bit by bit: the result is `1` only if **both** bits are `1` — like multiplication in algebra.

| a | b | a & b |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

**Example**: `5 & 3`

```text
  0101  (5)
& 0011  (3)
  ----
  0001  = 1
```

```c
#include <stdio.h>

int main(){
    int a = 5;      // 0101
    int b = 3;      // 0011
    int result = a & b; // 0001
    printf("5 & 3 = %d\n", result); // result = 1
    return 0;
}
```

## 2. OR (`|`)

The result is `1` if **either** bit is `1`.

| a | b | a \| b |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 1 |

**Example**: `5 | 3`

```text
  0101  (5)
| 0011  (3)
  ----
  0111  = 7
```

```c
#include <stdio.h>

int main(){
    int a = 3;      // 0011
    int b = 5;      // 0101
    int result = a | b; // 0111
    printf("3 | 5 = %d\n", result); // result = 7
    return 0;
}
```

## 3. XOR (`^`)

The result is `1` only when the two bits **differ**.

| a | b | a ^ b |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

**Example**: `5 ^ 3`

```text
  0101  (5)
^ 0011  (3)
  ----
  0110  = 6
```

```c
#include <stdio.h>

int main() {
    int a = 5;  // 0101
    int b = 3;  // 0011
    int result = a ^ b;
    printf("5 ^ 3 = %d\n", result); // 6
    return 0;
}
```

> **A crucial XOR property**: `a ^ b ^ b == a` — XORing with the same value twice restores the original. **This property is the foundation of the XOR encryption/decryption** used heavily in malware — review it carefully.

## 4. NOT (`~`)

Flips all bits (0 → 1 and 1 → 0).

**Example**: `~5`

```text
  00000101  (5)
~ --------
  11111010  = -6 in two's complement
```

```c
#include <stdio.h>

int main(){
    int a = 5;
    int result = ~a;
    printf("~5 = %d\n", result); // result = -6
    return 0;
}
```

> The result is `-6` because signed numbers are stored in two's complement. If number systems are unclear, review the accompanying number-systems material.

## 5. Left Shift `<<`

Moves bits left by a specified amount, filling the gaps with zeros. **Shifting by 1 equals multiplying by 2**.

```text
  0101  (5)  << 1
  ----
  1010  = 10
```

```c
#include <stdio.h>

int main(){
    int a = 5;
    printf("5 << 1 = %d\n", a << 1); // 10 (5 × 2)
    printf("5 << 2 = %d\n", a << 2); // 20 (5 × 4)
    return 0;
}
```

## 6. Right Shift `>>`

Moves bits right. **Shifting by 1 equals dividing by 2**.

```text
  1010  (10)  >> 1
  ----
  0101  = 5
```

```c
#include <stdio.h>

int main(){
    int a = 10;
    printf("10 >> 1 = %d\n", a >> 1); // 5 (10 / 2)
    printf("10 >> 2 = %d\n", a >> 2); // 2 (10 / 4)
    return 0;
}
```

## Compound Bitwise Assignment

| Operator | Meaning |
|---|---|
| `a &= b` | `a = a & b` |
| `a \|= b` | `a = a \| b` |
| `a ^= b` | `a = a ^ b` |
| `a <<= n` | `a = a << n` |
| `a >>= n` | `a = a >> n` |

```c
int a = 5;   // 0101
a &= 3;      // 0101 & 0011 = 0001
printf("%d\n", a); // 1

a = 5;
a |= 3;      // 0101 | 0011 = 0111
printf("%d\n", a); // 7

a = 5;
a ^= 3;      // 0101 ^ 0011 = 0110
printf("%d\n", a); // 6

a = 5;
a <<= 1;     // 0101 << 1 = 1010
printf("%d\n", a); // 10
```

## Operator Precedence — A Real Trap

> **Beware**: bitwise operators have **lower** precedence than comparison operators! This causes subtle bugs:
> ```c
> if (x & 3 == 1)      // parsed as x & (3 == 1) ← wrong!
> if ((x & 3) == 1)    // correct: with parentheses
> ```
> When reading code, always check the parentheses — this matters both for reading assembly and for spotting convoluted malicious code.

## Increment and Decrement

Unary operators that increase or decrease a variable's value by 1:

| Operator | Behavior |
|---|---|
| `++x` (prefix) | increments first, then uses the new value |
| `x++` (postfix) | uses the original value, then increments |
| `--x` / `x--` | same principle for decrement |

```c
int x = 5;
printf("%d\n", ++x); // 6 (increment first, then print)
x = 5;
printf("%d\n", x++); // 5 (print first, then increment)
printf("%d\n", x);   // 6 (after the increment)
```

### With Pointers

```c
int arr[] = {1, 2, 3, 4};
int *p = arr;
printf("%d\n", *p++);  // print 1, then advance the pointer to the next element
printf("%d\n", *p);    // print the second element = 2
```

> These patterns (`*p++`, `p++`) appear constantly in decryption loops in the assembly — you will see them countless times when analyzing malware.

## The Ternary Operator `?:`

C's only conditional operator — a compact alternative to if/else:

```c
Condition ? exp1 : exp2
```

If the condition is true `exp1` runs, otherwise `exp2`:

```c
#include <stdio.h>

int main(){
    int x = 10, y = 5;
    int max = (x > y) ? x : y; // max = 10
    printf("The bigger is: %d\n", max);
    return 0;
}
```

## The RE Connection — Why These Operators Are Essential

### 1. `& 0xFF` — Truncating a Byte

The most famous use: keeping only the last byte. **This is exactly what appeared in my Flare-On analysis**: `and eax, 0FFh` in the assembly.

```c
int value = 0x12345678;
int low_byte = value & 0xFF; // 0x78
```

### 2. `|` — Setting Flags

Setting specific bits without touching the rest:

```c
flags |= 0x01;  // enable the first flag
flags |= 0x80;  // enable the last flag
```

### 3. `^` — XOR Decryption (the Most Important in Malware)

This pattern appears in thousands of samples — decrypting data byte by byte:

```c
unsigned char encrypted[] = {0x6A, 0x75, 0x73, 0x74};
for (int i = 0; i < 4; i++) {
    encrypted[i] ^= 0x1F;  // decrypt with the key
}
```

And it appears in assembly as this pattern:
```asm
mov al, [esi]   ; read the encrypted byte
xor al, 0x1F    ; decrypt
mov [edi], al
inc esi         ; advance — here is *p++ / p++
```

### 4. `& 0x80` — Testing a Specific Bit

Testing a particular flag (like a sign bit):

```c
if (flags & 0x80) {
    // bit 7 is set
}
```

### 5. `>>` and `<<` — Multiplication and Division Arithmetic

Shifts appear in address calculations and buffer sizing — and sometimes the compiler uses them instead of `* 2` and `/ 2` as an optimization.

## Summary

Bitwise operators are **the shared language between C and assembly** — every `and`/`or`/`xor`/`shl`/`shr` you see in the disassembly is a direct application of what you learned here. Mastering them means you are reading code that the processor actually executes, not just program text.





