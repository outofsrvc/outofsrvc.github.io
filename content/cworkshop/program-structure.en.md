---
title: "Program Structure"
description: "The basic anatomy of a C program: includes, main, comments, global variables, and functions — and what each part leaves in the binary."
date: 2026-08-13T10:00:00+03:00
slug: "program-structure"
weight: 2
hex: "0x01"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "assembly"]
translationKey: "program-structure"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will look at the basic structural anatomy of a C program. The structure consists of essential parts that every C program adheres to:

```c
#include <stdio.h> // Including system libraries
// Definition of constants or global variables


// Other functions


int main(){
    // Core code here
    // Variables, functions, loops, conditionals
    return 0; // Return value to the operating system
}


// Other functions
```

## 1. Includes

Example: `#include <stdio.h>` — used to import built-in libraries. This particular library is responsible for input and output, and its name `stdio` stands for:

**St**an**d**ard **I**nput/**O**utput.

The `.h` extension stands for **header** — header files that declare functions before they are used.

> **An important point for RE**: these calls (such as `printf`, `scanf`) will later appear in the **Import Table** of the PE executable. When we see an imported function in the imports, we know the program uses it — and that is the first thread to pull in any analysis.

## 2. The main() Function

This is the starting point when any C program executes — the code inside this function runs first. Its common forms:

```c
int main(void)                  // no arguments
int main(int argc, char *argv[]) // with command-line arguments
```

- `argc`: the number of arguments.
- `argv`: an array of strings representing the arguments themselves.

> **An important point for RE**: the second form matters a great deal to you, because many programs are analyzed with command-line arguments — and those arguments will be found on the stack when entering `main` in the disassembly.

### About `return 0;`

`return 0;` returns a value to the operating system. A value of `0` means the program succeeded, while any non-zero value means failure or an error.

> Note: in modern C (C99+), if `main` ends without a `return`, the compiler implicitly assumes `return 0`.

## 3. Comments

Comments are added using `//` for a single line or `/* */` for multi-line blocks:

```c
// this is a single-line comment

/*
   this is a
   multi-line comment
*/
```

> **An important point for RE**: comments never appear in the assembly at all — they are removed during preprocessing (which we covered in the previous article). Any comment in the source code only helps the analyst as evidence of the programmer's original intent.

## 4. Global Variables

These are defined outside any function and are available to every function in the program. In RE you will notice they are stored in a dedicated section of the executable (static data such as `.data` and `.bss`) rather than on the stack — a fundamental difference between global and local variables, which we will detail in the assembly series.

## 5. Other Functions

Functions can be defined before or after `main`. If defined after `main`, they must be declared before it with a **prototype** so the compiler knows about them:

```c
int add(int a, int b); // prototype: declaration only

int main(){
    int result = add(3, 5);
    return 0;
}

int add(int a, int b){ // the actual definition
    return a + b;
}
```

## Hands-on Exercise

Write the following program and apply the compilation stages you learned in the previous article to it:

```c
#include <stdio.h>

int main(void){
    printf("Hello, RE!\n");
    return 0;
}
```

```bash
gcc -E hello.c -o hello.i    # watch #include expand into thousands of lines
gcc -S hello.i -o hello.s    # watch printf turn into a call
gcc -c hello.s -o hello.o
gcc hello.o -o hello
```

Notice how both `#include` and the comments disappear in the first stage, and how `call printf` appears in the assembly.
