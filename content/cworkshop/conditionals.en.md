---
title: "Conditional Statements"
description: "if/else, switch, and the ternary operator — and how they become cmp instructions, conditional jumps, and jump tables in disassembly."
date: 2026-08-13T10:00:00+03:00
slug: "conditionals"
weight: 6
hex: "0x05"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "assembly"]
translationKey: "conditionals"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will cover if / else-if / else, nested if statements, the switch statement, and the ternary operator.

**Conditional statements**: used to make decisions based on certain conditions — different instructions execute depending on whether a condition evaluates to true or false.

## 1. The if Statement

Used to check a condition and execute code if it is true. It is the simplest form.

```c
if (condition) {
    // runs if the condition is true
}
```

## 2. The if-else Statement

Executes one piece of code if the condition is true, and a different piece if it is false.

```c
if (condition) {
    // runs if the condition is true
} else {
    // runs if the condition is false
}
```

## 3. The if-else-if-else Statement

Used when there is more than one condition. Conditions are checked in sequence until one of them becomes true.

```c
if (condition1) {
    // runs if condition 1 is true
} else if (condition2) {
    // runs if condition 2 is true
} else {
    // runs if all conditions are false
}
```

### Example

```c
#include <stdio.h>

int main() {
    int grade = 85;

    if (grade >= 90) {
        printf("Excellent\n");
    } else if (grade >= 75) {
        printf("Good\n");
    } else {
        printf("Needs Improvement\n");
    }

    return 0;
}
```

## 4. Compound Conditions (`&&` and `||`)

Multiple conditions can be combined inside a single if:

```c
if (age >= 18 && hasID) {
    // runs only if both conditions hold (AND)
}

if (error == 0 || retry < 3) {
    // runs if at least one holds (OR)
}
```

## 5. Nested if Statements

Writing an if statement inside another if to check multiple conditions in sequence.

```c
if (condition1) {
    if (condition2) {
        // runs if both conditions are true
    }
}
```

### Example

```c
#include <stdio.h>

int main() {
    int age = 20;
    int hasID = 1;

    if (age >= 18) {
        if (hasID) {
            printf("You can enter.\n");
        } else {
            printf("You need an ID.\n");
        }
    } else {
        printf("You are underage.\n");
    }

    return 0;
}
```

## 6. The switch-case Statement

We use it to test a variable's value against a list of possibilities we define. If the value matches any possibility, the commands placed under that possibility run. Each possibility is called a `case`.

```c
switch (variable) {
    case value1:
        // runs if the variable equals value1
        break;
    case value2:
        // runs if the variable equals value2
        break;
    default:
        // runs if the value matched no case
}
```

### Example

```c
#include <stdio.h>

int main() {
    int day = 3;

    switch (day) {
        case 1:
            printf("Monday\n");
            break;
        case 2:
            printf("Tuesday\n");
            break;
        case 3:
            printf("Wednesday\n");
            break;
        default:
            printf("Invalid day\n");
    }

    return 0;
}
```

> **Every case needs `break`** — without it, execution continues into the following cases regardless of the value (this is called fall-through). Try removing `break` from the example and see the behavior for yourself.

> **Important note**: conditions in if or switch rely on boolean values:
> - `0` = false
> - non-`0` = true

## 7. The Ternary Operator

A shorthand for a simple if-else.

```c
condition ? true_expression : false_expression
```

### Example

```c
int a = 10, b = 20;
int max = (a > b) ? a : b;   // identical in meaning to the if-else below
```

This is exactly equivalent to:

```c
int max;
if (a > b) {
    max = a;
} else {
    max = b;
}
```

## What Do These Statements Look Like in Assembly?

This is the most important section of the article — because everything above will appear in the disassembly in these forms:

### if → `cmp` + a conditional jump

Every if statement becomes a `cmp` followed by a conditional jump (`jz`, `jnz`, `jg`, `jl`...). "Condition true/false" is precisely the state of the flags (`ZF`, `SF`, `OF`) set by the comparison — the same rules as "0 = false, non-0 = true" mentioned earlier.

```asm
cmp [grade], 90     ; grade >= 90 ?
jl  not_excellent   ; if not greater-or-equal, jump away
; ... Excellent branch
```

### switch → Jump Table

When a programmer writes a switch with several consecutive cases, the compiler may turn it into a **jump table** — an array of addresses stored in `.rdata`, indexed directly.

> **An essential RE skill**: seeing a jump table in the disassembly tells you instantly that the original code was a switch, and you can recover every case from the table itself. if-else chains, on the other hand, appear as sequences of consecutive comparisons — a decisive visual difference during analysis.

### break in switch → `jmp`

The `break` becomes an unconditional jump out of the switch — which is why a compiled switch looks like a pearl necklace (comparisons → internal jumps → single exit).

### Ternary → No Difference

The compiler treats the ternary exactly like an ordinary if-else — you will not see any distinctive shape for it in the assembly.

## Hands-on Exercise

1. Write the `grade` example next to the `switch` example — then inspect both with `gcc -S`: the first produces a chain of `cmp/jl`, while the second may produce a jump table (take it to IDA/Ghidra later to see the table).
2. Remove `break` from the switch and confirm the fall-through behavior.
3. Rewrite the ternary example as if-else and vice versa, then compare the resulting assembly — you will find them identical.
