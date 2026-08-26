---
title: "The Preprocessor"
description: "Preprocessor directives: macros, conditional compilation, and #pragma pack — and what each leaves behind in the binary."
date: 2026-08-13T10:00:00+03:00
slug: "preprocessor"
weight: 19
hex: "0x18"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "preprocessor", "macros", "assembly"]
translationKey: "preprocessor"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

After this series of articles we have completed the fundamentals of C — and next, God willing, comes the Malware Development series.

In this article we cover the **directives** of the preprocessor — the `#` sign followed by reserved words that the preprocessor understands, substituted before compilation.

> A recap of the compilation stages (covered in the first article) — in isolated form:
>
> ```bash
> gcc -E file.c -o file.i    # 1. Preprocessor
> gcc -S file.i -o file.s    # 2. Compiler
> gcc -c file.s -o file.o    # 3. Assembler
> gcc file.o -o file         # 4. Linker
> ```

## 1. Inclusion Directives

### `#include`

It enables restructuring: splitting a large program into multiple files, each responsible for a specific function:

```text
• i/o.c          — input and output
• processing.c   — processing
• storing.c      — storage
• retrieving.c   — retrieval
```

If a client requests modifying only the processing, you go to that file without touching the rest.

> **Two important notes:**
> 1. **Never** use absolute paths — like `#include "C:\\Users\\...\\Lab.h"` — because users' machines differ, and system-dependent paths break portability.
> 2. Relative paths are preferred — resolved according to the compiler's include path rules.

## 2. Macro Directives

### `#define` — Defining a Macro

It splits into two kinds:

### Object-Like Macro

Example: the circle area formula `Area = πr²`. The value of π (3.14) is a **magic number** — a mysterious figure that is hard to understand when reading code. The fix: give it a name:

```c
#define PI 3.14

a = PI * r * r;
```

After preprocessing it becomes:

```c
a = 3.14 * r * r;
```

**A practical example**: a program controlling a car's air conditioner based on temperature and geographic location:

```c
#define SHAM_HOT  35   // the Levant: above 35 degrees
#define RUSSIA_HOT 15  // Russia: above 15 degrees
```

**The benefits:**
1. Easy to modify at any time.
2. Highly readable code.
3. No magic numbers.
4. **Consumes no memory** — because substitution happens during preprocessing.

> Stylistically: `#define`s are written in **uppercase** to distinguish them from variables.

### Function-Like Macro

Ordinary functions in assembly go through stack frame construction, return address saving, and a call — consuming processing time. For tiny functions (like addition) we use a macro:

```c
#define Add(x, y) ((x) + (y))

int main(void) {
    int z = Add(3, 4); // after preprocessing: z = 3 + 4
    return 0;
}
```

A more complex example:

```c
#include <stdio.h>
#define MIN(a, b) ((a) < (b) ? (a) : (b))

int main(void) {
    printf("MIN between 10 and 20 is: %d\n", MIN(10, 20));
    return 0;
}
```

> Note the extra parentheses `((a) + (b))` — they prevent precedence errors at call sites (we covered operator precedence in the bitwise operators article).

### Multiline Macros

To continue a macro onto a new line, place a **backslash `\` as the last character of the line** (followed immediately by the newline):

```c
#define print() printf("not");\
             printf("bit")
```

Both lines are part of the macro.

### Predefined Macros

The language defines these automatically (in GCC/Clang):

| Macro | Purpose |
|---|---|
| `__DATE__` | current compilation date |
| `__TIME__` | current compilation time |
| `__FILE__` | the current file's name |
| `__LINE__` | the current line number |

> Beware: their names have **two underscores** at start and end (`__LINE__`, not `_LINE_`).

### `#undef` — Removing a Definition

When you need to remove a macro from a certain point onward:

```c
#define PI 3.14

{
    // code using PI
}

#undef PI  // from here on PI is undefined
```

## 3. Conditional Directives

### `#ifdef` / `#else` / `#endif`

Similar to ordinary conditionals, but resolved **at preprocessing time**, and must end with `#endif`:

```c
#include <stdio.h>
#define NOINPUT

int main(void) {
    int a = 0;
    int z = 5;

    #ifdef NOINPUT     // if NOINPUT is defined
    a = z;             // use the value directly
    #else
    printf("Enter a: ");
    scanf("%d", &a);
    #endif

    printf("value of a: %d\n", a);
    return 0;
}
```

### `#ifndef` — Preventing Multiple Inclusion

Executes if the identifier is not defined — its primary use is the include guard:

```c
#ifndef HEADER_FILE
#define HEADER_FILE
// file contents
#endif
```

## 4. Miscellaneous Directives

### `#error`

Stops compilation with an error at the preprocessing stage:

```c
#ifndef _MATH_H
#error "define MATH lib then compile"
#endif
```

The message next to `#error` is displayed.

### `#warning`

Emits a warning without stopping compilation:

```c
#warning "This is a warning"
```

### `#pragma` — Compiler Directives

Addresses the **compiler** itself, not the preprocessor:

```c
#pragma pack(push, 1)   // (see the RE section below)
```

> **Caution**: old compilers (like Turbo C/Borland) had `#pragma startup/exit` to change the entry point — **those are specific to those compilers and do not work in GCC/Clang**. In standard C, execution always starts at `main`.

## The RE Connection — Why the Preprocessor Is Essential

### 1. `#pragma pack(n)` — The Most Important for RE

It controls struct alignment — overriding the default padding with a specific alignment. **Essential when reading files or protocols stored without padding** (such as custom file formats):

```c
#pragma pack(push, 1)   // one-byte alignment — no padding
typedef struct {
    u16 magic;
    u32 size;
    u8  flags;
} file_header;
#pragma pack(pop)
```

> During analysis: a struct whose size **is not a multiple of 4** (say 9 bytes) almost certainly used pack(1).

### 2. Macros Vanish in the Binary

What you see as `MIN(a,b)` in the source appears in the disassembly as an **expanded ternary** (`cmp` + `jl`/`jge`) — no trace of the macro's name. This explains why you see duplicated code in asm instead of function calls.

### 3. `__DATE__` and `__TIME__` in Malware

Malware samples check their **build time** and compare it with the current time — a common technique for detecting VMs or detecting that the sample was run long after building. Seeing them in strings indicates potentially malicious behavior.

### 4. `#ifdef` and Conditional Builds

Used to separate build variants (Debug/Release, Windows/Linux). In the resulting binary you see **only one version** — and the analyst notices the absent paths and infers the compilation options used.

## Summary

The preprocessor is **the stage where all these choices are resolved before compilation** — its output (the `.i` file) is what the compiler actually sees. In RE, knowing what was resolved (macros, #ifdef, pack) explains strange shapes in the disassembly — from duplicated code to unexpected struct sizes.

---

God willing, this series has delivered what was required of it. If I erred, it is from myself; if I got it right, it is from Allah. Do not forget us in your prayers.

















