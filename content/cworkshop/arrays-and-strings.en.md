---
title: "Arrays and Strings"
description: "Arrays and strings in memory, the dangerous string.h functions, and their direct relationship to buffer overflow vulnerabilities."
date: 2026-08-13T10:00:00+03:00
slug: "arrays-and-strings"
weight: 8
hex: "0x07"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "memory", "assembly", "strings"]
translationKey: "arrays-and-strings"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will cover arrays and strings, their uses, and their importance in reverse engineering.

> From this article onward we go deeper — because the remaining articles are among the most valuable for reverse engineering, and mastering them is essential.

Arrays and strings are among the most important data structures for storing and processing data, and they differ in usage and structure. **Understanding how they live in memory is the foundation of executable analysis, especially when it comes to uncovering vulnerabilities.**

## 1. Arrays

**What is an array?** A collection of elements of the same type stored in **contiguous** memory locations. Each element is accessed by an index.

```c
data_type array_name[size];
// data_type: the element type (e.g. int, char, float)
// array_name: the array's name
// size: how many elements it can hold
```

### Examples

```c
int numbers[5];          // five integers
float prices[10];        // ten decimals
char letters[3];         // three characters
```

### Initialization

```c
int numbers[5] = {1, 2, 3, 4, 5};  // set every value
int numbers[5] = {1, 2};           // the rest are automatically zeroed
```

### Access and Modification

```c
int x = numbers[2];    // get the third element
numbers[0] = 10;       // modify the first element
```

> **A very important note**: the first element's index is `0` — `numbers[0]` is the first.

### Example

```c
#include <stdio.h>

int main() {
    int numbers[5] = {10, 20, 30, 40, 50};

    for (int i = 0; i < 5; i++) {
        printf("numbers[%d] = %d\n", i, numbers[i]);
    }

    return 0;
}
```

### Multidimensional Arrays

An array of rows and columns:

```c
int matrix[2][3] = {
    {1, 2, 3},
    {4, 5, 6}
};
```

Accessing an element:

```c
int x = matrix[1][2]; // second row, third column
```

## 2. Strings

**What is a string?** An array of characters terminated by the string-ending symbol `'\0'` (the null character). Remember it from the escape sequences article.

```c
char string_name[size];
```

### String Initialization

**1. Manually, character by character:**

```c
char name[4] = {'A', 'l', 'i', '\0'};
```

> Note that we allocated four slots: three characters + `\0`.

**2. As a literal:**

```c
char name[] = "Ali";  // \0 is added automatically here
```

### Accessing Characters

```c
char first_char = name[0]; // the first character
name[1] = 'e';             // modify the second character
```

### Example

```c
#include <stdio.h>

int main() {
    char name[] = "Ahmed";

    printf("Name: %s\n", name);

    for (int i = 0; name[i] != '\0'; i++) {
        printf("Character %d: %c\n", i, name[i]);
    }

    return 0;
}
```

## String Functions

These functions live in `<string.h>` — you need to master them because they appear constantly when analyzing binaries:

| Function | Purpose | Security note |
|---|---|---|
| `strlen` | string length | safe (read-only) |
| `strcpy` | copy a string | **unsafe — never checks the destination size** |
| `strcat` | concatenate two strings | **unsafe — never checks the destination size** |
| `strcmp` | compare two strings | safe (read-only) |

```c
#include <string.h>

int length = strlen(name);              // length
strcpy(destination, source);            // copy
strcat(string1, string2);               // concatenate
if (strcmp(string1, string2) == 0) {    // compare
    printf("Strings are equal.\n");
}
```

### Comprehensive Example

```c
#include <stdio.h>
#include <string.h>

int main() {
    char str1[20] = "Hello";
    char str2[20] = "World";

    strcat(str1, " ");
    strcat(str1, str2);
    printf("Concatenated String: %s\n", str1);

    if (strcmp(str1, "Hello World") == 0) {
        printf("Strings are equal.\n");
    } else {
        printf("Strings are not equal.\n");
    }

    printf("Length of str1: %zu\n", strlen(str1)); // %zu because strlen returns size_t

    return 0;
}
```

## Arrays vs Strings

| | Arrays | Strings |
|---|---|---|
| **Type** | any type (int, float, char...) | char only |
| **Terminator** | none | ends with `\0` |
| **Usage** | structured data (numbers, values...) | text |
| **In memory** | contiguous same-size elements | contiguous characters + `\0` |

## Why Arrays and Strings Matter in Reverse Engineering

### A. An Array in Assembly = Base Address + Offset

An array is never passed as a single object, but as **the address of its beginning**. The access `numbers[2]` becomes:

```asm
; numbers starts at address X, and int = 4 bytes
lea eax, [X]          ; base address
mov edx, [eax + 2*4]  ; numbers[2] = (address + offset)
```

The index turns into **arithmetic offset**: `index × element size`. That is why disassembly shows patterns like `[eax + ecx*4]` — this is the essence of working with arrays.

### B. The Direct Link to Buffer Overflow

A `char buf[10]` on the stack followed by `strcpy(buf, source)` — if the source exceeds 10 bytes, it **writes over whatever follows in memory** (EBP, then the return address).

> An array is not a "protected box" — it is a **run of bytes with no guard**. Understanding this is the gateway to buffer overflow vulnerabilities covered later in the exploitation series, and it is why `strcpy` and `strcat` are considered among the most dangerous functions in C (hence the safer `strncpy` and `strlcpy`).

### C. Strings Inside Binaries

- Strings are stored verbatim (ASCII/UTF) inside the executable in `.rdata` — which is why tools like `strings` can extract them easily.
- When an analyst finds a string in a binary, they locate it then follow the **cross-references (XREFs)** to see which function uses it — that is the first thread of any analysis.

### D. Tooling

Tools like IDA Pro and Ghidra analyze memory addresses and automatically identify arrays and stored strings — but understanding what happens **underneath** these tools is what separates a real analyst.

## Hands-on Exercise

1. Write `char buf[16]` and copy a longer string into it with `strcpy` (say 40 characters) — watch the program crash or behave strangely. These are the beginnings of hands-on overflow experimentation that you will expand on later.
2. Print the array's address with `%p` (`printf("%p", numbers)`) — you will see a memory address, the very same one you will see in the assembly.
3. Inspect your program in IDA/Ghidra and note the `[eax + ecx*4]` pattern used to access `numbers[i]`.


