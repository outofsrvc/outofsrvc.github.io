---
title: "Escape Sequences"
description: "Escape sequences and the null terminator (0x00): how strings are actually stored, and why strings inside binaries end with zero."
date: 2026-08-13T10:00:00+03:00
slug: "escape-sequences"
weight: 5
hex: "0x04"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "strings"]
translationKey: "escape-sequences"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

In this article we will look at escape sequences, their importance, and their uses.

## What Are Escape Sequences?

They are used inside strings to represent special characters or to control output formatting. They are written as a backslash (`\`) followed by one or more characters. Their purpose is to make it easy to represent special characters that are hard to type inside a string literal — such as tab or newline.

## 1. Controlling Formatting and Output

### `\n` — Newline

```c
printf("Hello\nWorld");
```

Output:

```text
Hello
World
```

### `\t` — Tab

```c
printf("Hello\tWorld");
```

Output:

```text
Hello   World
```

### `\r` — Carriage Return

```c
printf("Hello\rWorld");
```

> Note: on Windows a new line is usually written as the pair `\r\n` (CRLF). This distinction matters in RE when analyzing text files and protocols.

### `\b` — Backspace (move back one step)

```c
printf("Hello\bWorld");
```

Output: the cursor moves back over the last `o` and `W` is written in its place, so it displays `HellWorld`.

## 2. Inserting Special Symbols

### `\'` — Single Quote

```c
printf("It\'s cool!");
```

### `\"` — Double Quote

```c
printf("He said, \"Hello!\"");
```

### `\\` — The Backslash Itself

```c
printf("C:\\Windows\\System32");
```

## 3. Inserting Characters by Numeric Code

### `\ooo` — Representing a Character in Octal

```c
printf("\101");
```

> `\101` in octal = 65 in decimal = the character `A`.

### `\xhh` — Representing a Character in Hexadecimal

```c
printf("\x41");
```

> `\x41` = 65 = the character `A`. Both results are identical — the only difference is the number system.

## 4. The Most Important One for Reverse Engineering: `\0` (The Null Terminator)

```c
char str[] = "Hello\0World";
printf("%s", str);
```

**The output is `Hello` only** — because `printf` stops at the first zero.

> ### Why is this the most important concept in the entire article?
>
> **Strings in C do not store their length — they end with a null character (0x00)**. Any string is read until a zero is encountered.
>
> This explains:
> - Why strings you see when opening a PE file in HxD end with `00 00`.
> - Why strings print partially if they contain a `\0` in the middle.
> - Why binary analysts handle strings with care — the analyst extracts consecutive printable characters and determines where each string ends by finding the zero.
> - Why exploit payloads are crafted with extreme care around string boundaries — overflowing them is the root of many vulnerabilities (buffer overflow).

## Comprehensive Example

```c
#include <stdio.h>

int main() {
    printf("Escape Sequences Demo:\n");
    printf("Newline -> \\n\n");
    printf("Tab -> \\t\tTabbed Text\n");
    printf("Backslash -> \\\\\n");
    printf("Double Quote -> \\\"Hello\\\"\n");
    printf("Octal -> \\101\n");
    printf("Hexadecimal -> \\x41\n");
    return 0;
}
```

> Note in the example: writing `\\n` inside the string prints the characters `\n` literally, while `\n` on its own creates a new line.

## Hands-on Exercise

1. Write a program that prints `A` three ways: directly, via `\101`, and via `\x41` — and confirm all three outputs match.
2. Open the resulting executable in HxD and look for the value `41` that is stored — you will find the character ended up in the file as an actual value, because escape sequences are processed during compilation and do not remain as text.
3. Print `Hello\0World` and `Hello World` and observe the difference in how reading ends — this is exactly the null-terminated string behavior.

> **Reminder**: practice every sequence hands-on. Do not fear forgetting them — with practical repetition they will feel easy; they only require a little drilling. May Allah grant you success.

