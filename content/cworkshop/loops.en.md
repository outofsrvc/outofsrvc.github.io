---
title: "Loops"
description: "while, do-while, for, and nested loops — and the fingerprint that gives a loop away in assembly: the backward jump."
date: 2026-08-13T10:00:00+03:00
slug: "loops"
weight: 7
hex: "0x06"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "assembly"]
translationKey: "loops"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will cover loops: `while`, `do-while`, `for`, and nested loops — along with how to control them and when each one fits.

**Loops**: used to execute a block of instructions repeatedly based on a condition. There are several kinds, and each suits a different use case.

## 1. The while Loop

Used when repetition depends on a condition — and the loop may not run at all if the condition is never met.

```c
while (condition) {
    // runs as long as the condition holds
}
```

### Example

```c
#include <stdio.h>

int main() {
    int i = 1;

    while (i <= 5) {
        printf("Iteration %d\n", i);
        i++; // increment i
    }

    return 0;
}
```

> In a while loop the condition is checked **before** every iteration — if it fails from the start, the loop never runs.

## 2. The do-while Loop

Similar to while, but it **executes the code at least once** even if the condition is false from the start — the condition is checked **after** running the code.

```c
do {
    // runs at least once
} while (condition);
```

### Example

```c
#include <stdio.h>

int main() {
    int i = 1;

    do {
        printf("Iteration %d\n", i);
        i++;
    } while (i <= 5);

    return 0;
}
```

> **Note**: even if `i = 6` at the start, the code executes once before the condition is checked.

## 3. The for Loop

Used when the number of iterations is **known in advance**. It has three parts:

1. **Initialization** — the counter's initial definition.
2. **Condition** — the stop condition.
3. **Update** — modifying the variable after each iteration.

```c
for (initialization; condition; update) {
    // runs on every iteration
}
```

### Example

```c
#include <stdio.h>

int main() {
    for (int i = 1; i <= 5; i++) {
        printf("Iteration %d\n", i);
    }

    return 0;
}
```

## 4. Nested Loops

An outer loop wrapping an inner loop. The inner loop is part of the outer loop's body and completes a full cycle for every single iteration of the outer one.

```c
#include <stdio.h>

int main() {
    for (int i = 1; i <= 3; i++) {
        for (int j = 1; j <= 2; j++) {
            printf("Outer: %d, Inner: %d\n", i, j);
        }
    }

    return 0;
}
```

> Output: 3 outer iterations × 2 inner iterations = 6 lines.

## Controlling Loops

### break

Exits the loop immediately.

```c
#include <stdio.h>

int main() {
    for (int i = 1; i <= 10; i++) {
        if (i == 5) {
            break; // exit the loop
        }
        printf("%d\n", i);
    }
    return 0;
}
```

Output: `1 2 3 4` — the loop stopped at 5.

### continue

Skips the remaining instructions in the current iteration and moves to the next one.

```c
#include <stdio.h>

int main() {
    for (int i = 1; i <= 5; i++) {
        if (i == 3) {
            continue; // skip this iteration
        }
        printf("%d\n", i);
    }
    return 0;
}
```

Output: `1 2 4 5` — only the printing of 3 was skipped.

## Comparison Table

| Loop | When to use | Check | Executions |
|---|---|---|---|
| `while` | repetition driven by a changing condition | before every iteration | zero or more |
| `do-while` | the body must run at least once | after every iteration | one or more |
| `for` | iteration count known in advance | before every iteration | zero or more |

## What Do Loops Look Like in Assembly?

Continuing from the previous article — loops are **backward jumps** in the disassembly:

### The Typical Structure

Any loop becomes:

```asm
loop_start:
    ; loop body
    inc [i]        ; update (i++)
    cmp [i], 5     ; condition
    jle loop_start ; jump backward while the condition holds
```

The backward jump (the loop target) is the **clear fingerprint** that lets you spot a loop in the disassembly instantly.

### while vs do-while in Assembly

- `while`: the condition is checked **before** the body — the conditional jump sits at the start of the structure.
- `do-while`: the condition is checked **after** the body — the jump sits at the end.

> **An optimization fact**: the do-while structure is faster for the processor because it needs one jump. For that reason, at higher optimization levels (`O2`) the compiler may rotate a while loop into a do-while (loop rotation) — meaning the final assembly does not always match what you wrote.

### break and continue in Assembly

- `break` → a jump to the loop's final exit.
- `continue` → a jump straight to the next step (the update/condition).

### An Example from Malware Analysis

In real samples you will encounter loops that inspect data byte by byte (as in strlen or decryption routines) — they look like this:

```asm
decrypt_loop:
    mov al, [esi]   ; read a byte
    xor al, 0x1F    ; decrypt it
    mov [edi], al
    inc esi         ; advance
    inc edi
    cmp al, 0       ; end of string?
    jne decrypt_loop
```

This mental picture (read → process → advance → check → jump back) will serve you greatly when analyzing binaries.

## Hands-on Exercise

1. Rewrite the `for` example as a `while` and vice versa, then compare the generated assembly with `gcc -S` — you will notice the compiler builds both with the same structure.
2. Try `gcc -O2 -S` on a while loop — observe how it may turn into a do-while structure.
3. Write a loop that decrypts byte by byte (like the example above) and inspect it in IDA/Ghidra — it will match the structure shown in the "What Do Loops Look Like" section.

