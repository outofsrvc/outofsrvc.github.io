---
title: "Instruction Set Architecture"
description: "An introduction to the Instruction Set Architecture: machine instruction characteristics and their x86 formats and lengths, operand types, data types from Byte to QuadWord, and operation categories — arithmetic, logic, data transfer, and control flow."
date: 2026-08-25T11:00:00+03:00
slug: "comp-arch-0x2"
translationKey: "comp-arch-0x2"
weight: 3
hex: "0x2"
categories: [computer-architecture]
tags: [isa, assembly, x86, machine-code, opcodes, endianness, basics]
ShowToc: true
TocOpen: false
draft: false
---

بسم الله

Having taken an overview of the computer's structure in the [first article](../comp-arch-0x0/), and dived into what happens inside the processor in the [second one](../comp-arch-0x1/), today we meet the theoretical foundation linking binary instructions to the operations the processor executes. This article also serves as a practical introduction to Assembly, explaining the structure of its code. Here are the topics we will cover:

## Instruction Set Architecture (ISA)

The Instruction Set Architecture (**ISA**) is the theoretical foundation connecting machine code to the instructions the CPU executes:

![Abstraction layers: software above the ISA, hardware below](/assets/img/computer-arch-org/computer-arch-0x2/isa-abstraction.png)
_Figure (1): Where the ISA sits between software and hardware_

The ISA defines: the instructions supported by the processor, data types, registers, how memory is accessed, addressing modes, and other fundamental details.

In other words, the ISA is the language of communication between software and hardware — without descending into the fine details of the internal design (the microarchitecture).

## Machine Instruction Characteristics

A machine instruction consists of two parts:

**• Opcode:**
Determines the kind of action to perform — the command itself (`mov`, `add`, etc.).

**• Operands:**
Refer to the target data the action applies to. Each instruction is stored in memory as a sequence of bytes.

### Essential Facts

**Instruction format and length:**
As mentioned, every instruction consists of an opcode and operands; on x86, instruction lengths vary from 1 to 15 bytes.

**Operand count:**
x86 instructions allow zero to three operands, and most opcodes take two: a source and a destination. In Intel syntax the destination comes first, then the source. For example:

```asm
mov  eax, ebx
```

has two operands: the register `eax` — the destination that will receive the value — and the register `ebx` — the source the value is read from.

**Byte ordering (Endianness):**
x86 uses Little Endian ordering: the least significant byte is stored first in memory. The value `0x12345678` is laid out in memory as `78 56 34 12`.

**Example:**

```asm
mov  eax, [a]     ; move the value of variable a into EAX
add  eax, [b]     ; EAX now holds a, so add the value of variable b
mov  [sum], eax   ; move the resulting value from EAX into sum
```

## Types of Operands

An operand in machine code is whatever refers to the data being processed, and it can be one of the following:

**1. Immediate:**
A numeric constant embedded directly in the instruction itself:

```asm
mov  ebx, 5       ; move the constant 5 into EBX
```

**2. Register:**
The operand may be a processor register:

```asm
mov  eax, ebx     ; move the contents of EBX into EAX
```

**3. Memory address:**
The operand may be a memory address:

```asm
mov  eax, [ebx]   ; move the value stored at the address held in EBX into EAX
```

**Combined example:**

```asm
mov  eax, [a]     ; here a (memory) and eax (register)
add  eax, [b]     ; b (memory)
mov  [sum], eax   ; sum (memory)
```

## x86 Data Types

The x86 processor supports several data types, the most important being:

- **Byte:** the fundamental unit of data, 8 bits.
- **Word:** 16 bits.
- **DoubleWord:** 32 bits.
- **QuadWord:** 64 bits.

**Example:** suppose we have this C code:

```c
char c = 'A';
short s = 1000;
int i = 100000;
```

Here is how their sizes map onto assembly registers:

```asm
mov  al, 'A'       ; AL is an 8-bit subdivision of AX
mov  ax, 1000      ; AX is the 16-bit half of EAX
mov  eax, 100000   ; the full 32-bit EAX
```

## Types of Operations

Broadly speaking, there are fundamental operations from which virtually any code is composed — you could say about 90% of any program consists of these opcodes:

**• Arithmetic & Logic:**
Includes `add`, `sub`, `mul`, and `div`, plus the logic operations `not`, `and`, `or`, and `xor`. These instructions modify values in registers or memory.

**• Data Transfer:**
Instructions that move data between registers, memory, or input/output. The most famous is `mov`, which carries a value from a source (register, memory, or immediate) to a destination (register or memory). We also have `push` and `pop` for manipulating the stack.

**• Control Flow:**
Instructions controlling program execution: unconditional jumps (`jmp`), conditional jumps based on comparisons such as `cmp` paired with `je`/`jne`/`jle` and friends, plus function-call instructions `call` and `ret`. These rely on comparing values and changing the Instruction Pointer.

**• Input/Output (I/O):**
x86 has dedicated instructions for dealing with input and output devices, such as `in` and `out`, allowing reads and writes to device ports.

**Example:** suppose we have this C code:

```c
int a = 10, b = 3, c;                 // declare variables a, b, c
if (a > b) c = a - b; else c = 0;     // if a > b then c = a - b, else c = 0
```

This is its representation in assembly:

```asm
mov  eax, [a]
cmp  eax, [b]      ; compare a with b
jle  L_else        ; if a <= b, jump to the else branch
sub  eax, [b]      ; otherwise execute: a = a - b
mov  [c], eax
jmp  L_end         ; skip the else branch once done
L_else:
mov  dword [c], 0  ; else case: c = 0
L_end:
```

Notice how the `if` condition becomes a `cmp` instruction that sets the flags, followed by a conditional jump `jle` that inspects them: if the condition fails (a <= b), execution jumps to the `else` branch; if it holds, the subtraction runs and an unconditional `jmp` hops over the `else` branch.

## Summary

By understanding the Instruction Set Architecture — knowing that a machine instruction consists of an opcode and operands, and the types of each — you become able to translate binary code into assembly and follow how a program moves step by step.

Keep us in your prayers.
