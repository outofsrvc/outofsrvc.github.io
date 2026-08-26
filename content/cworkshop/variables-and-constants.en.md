---
title: "Variables and Constants"
description: "Declaring variables and naming rules, constants via const and #define, and the fundamental difference between them at compile time and in memory."
date: 2026-08-13T10:00:00+03:00
slug: "variables-and-constants"
weight: 3
hex: "0x02"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "memory"]
translationKey: "variables-and-constants"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will cover how to declare variables, the steps involved and the naming rules, plus how to define constants, their types, and how to declare them — and finally we will clarify the difference between variables and constants.

## First: Variables

**Definition**: a location reserved in memory (RAM) to store a specific value while the program runs.

> **A subtle note**: when a function ends, a local variable's space on the stack is reclaimed — but the value itself may **remain in physical memory** until it is overwritten by other values. This concept matters greatly in reverse engineering: analysts sometimes hunt for stale values that were never deliberately wiped.

### Steps for Declaring a Variable

1. Specify the **data type** (we will cover it in the next article, God willing).
2. **Name the variable**: the name must be unique and follow the naming rules below.
3. **Assign an initial value**: an optional step.

### Declaration Syntax

```c
dataType varName = value;
```

### Variable Naming Rules

- The name must start with a letter or `_` (underscore).
- It may contain letters (a-z, A-Z), digits (0-9), and `_`.
- Reserved keywords are not allowed (such as `int`, `return`).
- Case sensitivity: `Age` and `age` are two different variables.

### Example

```c
#include <stdio.h>

int main() {
    int age = 25;           // an int variable
    float height = 5.8;     // a decimal variable
    char grade = 'A';       // a character variable
    char name[] = "Ali";    // a string variable (array of characters)

    printf("Age: %d\n", age);
    printf("Height: %.1f\n", height);
    printf("Grade: %c\n", grade);
    printf("Name: %s\n", name);

    return 0;
}
```

Output:

```text
Age: 25
Height: 5.8
Grade: A
Name: Ali
```

## Second: Constants

**Definition**: similar to variables in their uses, but opposite in their most important property — their value cannot change during program execution.

### Types of Constants

| Type | Description | Example |
|---|---|---|
| Numeric | integers or decimals | `10`, `-5`, `3.14` |
| Character | a single character in single quotes | `'A'`, `'b'` |
| String | text in double quotes | `"Hello"` |
| Symbolic | defined with `const` or `#define` | `MAX_SIZE` |

### Two Ways to Define Constants

#### 1. Using `const`

```c
const dataType constName = value;
```

> **Important note**: a value must be present at the point of definition.

#### 2. Using `#define`

```c
#define constName value
```

### Example

```c
#include <stdio.h>

#define MAX_SIZE 100  // a symbolic constant using #define

int main() {
    const float PI = 3.14;  // a constant using const

    printf("Value of PI: %.2f\n", PI);
    printf("Maximum Size: %d\n", MAX_SIZE);

    // attempting to change a constant causes an error:
    // PI = 3.14159;     // the compiler forbids writing to const directly
    // MAX_SIZE = 200;   // #define is not even a variable!

    return 0;
}
```

Output:

```text
Value of PI: 3.14
Maximum Size: 100
```

## `const` vs `#define`

| | `const` | `#define` |
|---|---|---|
| **Definition** | a read-only variable whose value cannot change at runtime | a macro replaced textually during compilation |
| **Processing** | runtime | compile time |
| **Scope** | follows variable scoping rules (local/global) | no scope — substituted everywhere it is used |
| **Type** | requires specifying a type (safer) | typeless (can cause errors) |

> **An important point for RE**:
> - `#define` is replaced before compilation — you will see its value embedded directly inside instructions (immediates) or in `.rodata`/`.rdata`, with **no trace of it as a variable**.
> - A `const` is actually stored in memory like any other variable; the compiler merely prevents writing to it directly.
> - **Key takeaway**: `const` in C is **not a true compile-time constant** — it is a read-only variable, and it can in fact be modified through pointers (which is exactly what matters to an analyst who sees code writing to `const` memory).
> - True compiled-in constants come from `#define` or `enum`.

## Variables vs Constants

| | Variables | Constants |
|---|---|---|
| **Value** | can change during execution | cannot change during execution |
| **Storage** | on the stack or heap (dynamic data) | embedded in instructions or `.rodata`/`.rdata` |
| **Declaration** | `int x = 5;` | `const int x = 5;` or `#define X 5` |
| **Usage** | for values that change | for values fixed within the program's context |

## Hands-on Exercise (covering the previous articles together)

Apply the steps yourself and watch the real difference:

```c
#define LIMIT 100

int main(void){
    const int x = 42;
    int y = LIMIT;
    return 0;
}
```

```bash
gcc -E test.c -o test.i    # watch LIMIT vanish, replaced by 100 everywhere
gcc -S test.i -o test.s    # watch the values 42 and 100 appear in mov instructions
```

Note that `LIMIT` disappears completely after preprocessing, while `x` remains a real variable in the assembly.

---

> **Closing note**: understanding without practice is useless. Try every example with your own hands and the differences will stick in your mind. Your feedback matters — whether criticism to improve or encouragement to continue.

