---
title: "The Stages Code Goes Through"
description: "The four stages that turn C code into an executable — preprocessor, compiler, assembler, linker — and how each one shapes the binary."
date: 2026-08-13T10:00:00+03:00
slug: "compilation-stages"
weight: 1
hex: "0x00"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "compiler", "assembly", "basics"]
translationKey: "compilation-stages"
ShowToc: true
TocOpen: false
draft: false
---

Assalamu alaikum wa rahmatullahi wa barakatuh.

This series covers the fundamentals of the C language that matter most for **reverse engineering**. We will not expand into every detail of the language — only into what matters when reading disassembly and analyzing binaries.

## The Stages C Code Goes Through

We start with the **source code** — the `.c` file containing what we wrote. This code passes through four stages before it executes, and understanding them is extremely important:

### 1. The Preprocessor

Several things happen at this stage, most notably:

- Comment removal.
- Macro expansion (`#define`).
- Header expansion (`#include`).
- **Conditional compilation** (`#ifdef`, `#ifndef`, `#endif`) — very common in real-world code, and you will run into it constantly during analysis.

The output file has the `.i` extension.

### 2. The Compiler

This stage converts the code into assembly (**Assembly**), producing a file with the `.s` extension.

> **An important point for RE**: this is exactly the stage where **compiler optimizations** (`O0`/`O1`/`O2`/`O3`) are applied. These optimizations completely change the shape of the resulting disassembly — code compiled with `O0` looks radically different from code compiled with `O2`. Knowing which optimization level was used is essential when analyzing any binary.

### 3. The Assembler

At this stage the assembler converts the assembly code into machine language (zeros and ones), producing a file with the `.o` extension.

**Important note**: this file is a **relocatable object file** — meaning the addresses inside it are not final yet, and the calls are still unresolved. That is exactly why the fourth stage exists.

### 4. The Linker

This stage does the following:

- Merges all `.o` files together along with libraries.
- Binds function calls to their definitions.
- Resolves the final addresses.

The end result is an executable file.

> **An OS-dependent note**: on **Windows** the output is a PE with the `.exe` extension, while on **Linux** it is an ELF **with no extension** (such as `a.out`). This distinction is fundamental in reverse engineering, and we will cover it in detail later.

## Hands-on Exercise

It is important to learn how to perform these steps manually and watch the code transform from one stage to the next. On Linux using `gcc`:

```bash
gcc -E program.c -o program.i    # 1. Preprocessing
gcc -S program.i -o program.s    # 2. Compiler → Assembly
gcc -c program.s -o program.o    # 3. Assembler → Object file
gcc program.o -o program         # 4. Linker → Executable
```

Inspect the contents of each file at each stage and observe the differences.

### A Useful Tool: Compiler Explorer (godbolt)

[godbolt.org](https://godbolt.org/) shows you the generated assembly directly as you write the code, with the ability to choose the optimization level (`O0`/`O2`...). An indispensable tool for linking source code with what the compiler produces.

## Why Does This Matter in Reverse Engineering?

Every one of these stages leaves behind a trail that is useful to the analyst:

| Stage | The trail that matters to the analyst |
|---|---|
| Preprocessing | Seeing the code after macro expansion and understanding the real structure. |
| Compilation | Understanding how optimizations affect the shape of the assembly. |
| Assembly | Reading machine language and opcodes. |
| Linking | Understanding the executable's structure (PE/ELF) and external calls (imports). |

In short: understanding these four stages is the cornerstone of reading any binary, and it is what the rest of the series will build on, God willing.
