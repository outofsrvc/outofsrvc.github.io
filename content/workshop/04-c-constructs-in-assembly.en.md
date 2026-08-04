---
title: "Recognizing C Code Constructs in Assembly"
description: "How C constructs - variables, loops, conditionals, switches, arrays, structs and linked lists - look in assembly."
date: 2026-03-19T10:00:00+03:00
slug: "c-constructs-in-assembly"
weight: 4
hex: "0x4"
stage: "deep-dive"
categories: [deep-dive, programming]
tags: [reverse-engineering, malware-analysis, c-lang, x86-assembly]
translationKey: "c-and-asm"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah

When beginning to work with Assembly language, one invariably perceives a degree of difficulty owing to its low-level nature, in contrast to higher-level languages such as C or Python. In this research, we attempt to simplify assembly code so that it may be more readily understood.

The premise of this article is that we will author a simple program in C, compile it into x86 Assembly, and then explain the operations that occur in order to analyze the control flow.

> Before proceeding, if you do not possess sufficient familiarity with C and Assembly, we recommend reviewing the following series:
> 🔗 [C language series on the Shell Network](https://sh3ll.cloud/xf2/threads/4059/)
> 🔗 [Assembly language series on the Shell Network](https://sh3ll.cloud/xf2/threads/4305/)

---

## 1. Global vs. Local Variables

### Global Variables

We begin by authoring the following simple C code:

```c
int x = 1;
int y = 2;

void main() {
    x = x + y;
    printf("Total = %d\n", x);
}
```

We then compile it using the following command:

```bash
gcc -S -masm=intel -m32 -O0 filename.c -o filename.s
```

When we open the file bearing the `.s` extension, the following is produced:

**x86 Assembly:**

```assembly
00401003  mov eax, dword_40CF60
00401008  add eax, dword_40C000
0040100E  mov dword_40CF60, eax      ; [1] Store the result
00401013  mov ecx, dword_40CF60
00401019  push ecx
0040101A  push offset aTotalD        ; "total = %d\n"
0040101F  call printf
```

In assembly, global variables are expressed as memory addresses (Memory Addresses) such as: `dword_40CF60`.

### Local Variables

With respect to local variables, these are expressed as an offset relative to `ebp`, `esp`, or any other register (for example: `dword ptr [ebp-4]`). When we employ a disassembler such as IDA Pro (which we will examine in detail in forthcoming articles, God willing), the local variables appear clearly.

```c
void main()
{
int x = 1;
int y = 2;
x = x+y;
printf("Total = %d\n", x);
}
```

**x86 Assembly:**

```assembly
00401006  mov dword ptr [ebp-4], 1      ; [1]
0040100D  mov dword ptr [ebp-8], 2      ; [2]
00401014  mov eax, [ebp-4]
00401017  add eax, [ebp-8]
0040101A  mov [ebp-4], eax
0040101D  mov ecx, [ebp-4]
00401020  push ecx
00401021  push offset aTotalD ; "Total = %d\n"
00401026  call printf
```

> When we employ a disassembler such as `ida` (which we will examine in forthcoming articles, God willing), the local variables appear as illustrated here.

```assembly
00401006  mov [ebp+var_4], 1        ; [1]
0040100D  mov [ebp+var_8], 2        ; [2]
00401014  mov eax, [ebp+var_4]
00401017  add eax, [ebp+var_8]
0040101A  mov [ebp+var_4], eax
0040101D  mov ecx, [ebp+var_4]
00401020  push ecx
00401021  push offset aTotalD ; "Total = %d\n"
00401026  call printf
```

---

## 2. If Statement

```c
int x = 1;
int y = 2;

if(x == y) {
    printf("x equals y.\n");
} else {
    printf("x is not equal to y.\n");
}
```

**x86 Assembly:**

```assembly
00401006  mov [ebp+var_8], 1
0040100D  mov [ebp+var_4], 2
00401014  mov eax, [ebp+var_8]
00401017  cmp eax, [ebp+var_4]       ; [1] Comparison instruction
0040101A  jnz short loc_40102B       ; [2] Conditional jump to else
0040101C  push offset aXEqualsY_     ; "x equals y.\n"
00401021  call printf
00401026  add esp, 4
00401029  jmp short loc_401038       ; [3] Unconditional jump to skip else
0040102B loc_40102B:                 ; else section
0040102B  push offset aXIsNotEqualToY ; "x is not equal to y.\n"
00401030  call printf
```

The first element we encounter is the `CMP` instruction, followed by a conditional jump. (If the conditional jump is taken, the code has followed the `else` path.) If it is not taken, execution continues and performs an unconditional jump, which constitutes the primary `if` path.

---

## 3. For Loop

```c
int i;
for(i = 0; i < 100; i++) {
    printf("i equals %d\n", i);
}
```

**x86 Assembly:**

```assembly
00401004  mov [ebp+var_4], 0         ; [1] Initialization (i=0)
0040100B  jmp short loc_401016       ; [2] Jump to comparison
0040100D loc_40100D:                 ; Update region
0040100D  mov eax, [ebp+var_4]       ; [3]
00401010  add eax, 1                 ; Increment counter
00401013  mov [ebp+var_4], eax       ; [4]
00401016 loc_401016:                 ; Condition region
00401016  cmp [ebp+var_4], 64h       ; [5] 64h = 100
0040101A  jge short loc_40102F       ; [6] Conditional jump to exit loop
0040101C  mov ecx, [ebp+var_4]       ; Loop body
0040101F  push ecx
00401020  push offset aID            ; "i equals %d\n"
00401025  call printf
0040102A  add esp, 8
0040102D  jmp short loc_40100D       ; [7] Return to update
```

1. Initialization is performed with a local variable inside the `for`.
2. We observe an unconditional jump that leads to the `CMP` comparison.
3. This is followed by a conditional jump to exit. If the exit is not taken, the instructions within the `for` are executed.
4. Finally, an unconditional jump leads back to the update location (`increment` or `decrement`).
5. The process repeats until the exit condition is satisfied.

![For Loop in Graph](/assets/img/workshop/c-and-asm/graph-for-loop.png)
_Figure (1): An excerpt from IDA Pro._

---

## 4. While Loop

```c
int status = 0;
int result = 0;

while(status == 0) {
    result = performAction();
    status = checkResult(result);
}
```

**x86 Assembly:**

```assembly
00401036  mov [ebp+var_4], 0
0040103D  mov [ebp+var_8], 0
00401044 loc_401044:                 ; Loop start
00401044  cmp [ebp+var_4], 0         ; Comparison
00401048  jnz short loc_401063       ; [1] Conditional jump to exit
0040104A  call performAction
0040104F  mov [ebp+var_8], eax
00401052  mov eax, [ebp+var_8]
00401055  push eax
00401056  call checkResult
0040105B  add esp, 4
0040105E  mov [ebp+var_4], eax       ; Update status
00401061  jmp short loc_401044       ; [2] Return to loop start
```

This is similar to the `for` loop but somewhat simpler. It begins with a `CMP` comparison followed by a conditional jump to exit the `while`. If the jump is not taken, the code inside the loop is executed, and at its conclusion an unconditional jump repeats the cycle. This continues until the `CMP` condition is satisfied and execution jumps out, ending the loop.

---

## 5. Switch Statement

A `switch` may be compiled in one of two ways depending on the compiler:

### Method 1: If Style

This is nearly identical to chained `if` statements. We observe a number of comparisons equal to the number of cases, and after each comparison a conditional jump to execute the corresponding block. At the end, an unconditional jump leads to `default`.

```c
switch(i) {
    case 1: printf("i = %d", i+1); break;
    case 2: printf("i = %d", i+2); break;
    case 3: printf("i = %d", i+3); break;
    default: break;
}
```

**x86 Assembly:**

```assembly
00401013  cmp [ebp+var_8], 1
00401017  jz short loc_401027        ; [1] Case 1
00401019  cmp [ebp+var_8], 2
0040101D  jz short loc_40103D        ; Case 2
0040101F  cmp [ebp+var_8], 3
00401023  jz short loc_401053        ; Case 3
00401025  jmp short loc_401067       ; [2] Default

00401027 loc_401027:                 ; Execute Case 1
00401027  mov ecx, [ebp+var_4]       ; [3]
0040102A  add ecx, 1
; ... (remaining instructions) ...
```

![If Style in Graph](/assets/img/workshop/c-and-asm/graph-if-style.png)
_Figure (2): An excerpt from IDA Pro._

### Method 2: Jump Table

```c
switch(i) {
    case 1: printf("i = %d", i+1); break;
    case 2: printf("i = %d", i+2); break;
    case 3: printf("i = %d", i+3); break;
    case 4: printf("i = %d", i+4); break;
    default: break;
}
```

**x86 Assembly:**

```assembly
00401016  sub ecx, 1                 ; Subtract 1 because the compiler starts from 0
00401019  mov [ebp+var_8], ecx
0040101C  cmp [ebp+var_8], 3         ; Compare against maximum
00401020  ja short loc_401082        ; If out of range, go to default
00401022  mov edx, [ebp+var_8]
00401025  jmp ds:off_401088[edx*4]   ; [1] Direct jump via the table

; --- Target addresses ---
0040102C loc_40102C:
; ...
00401042 loc_401042:
; ...
00401082 loc_401082:                 ; End (Default/Exit)
00401082  xor eax, eax
00401087  retn

; --- Jump table ---
00401088 off_401088:                 ; [2]
00401088  dd offset loc_40102C
0040108C  dd offset loc_401042
00401090  dd offset loc_401058
00401094  dd offset loc_40106E
```

The jump-table method relies on subtracting a value to obtain a zero-based `index`, then comparing it against the number of cases. If it falls within range, it jumps directly using the table according to the equation `[edx*4]`. If it is outside the range, `default` is executed immediately.

![Jump Style in Graph](/assets/img/workshop/c-and-asm/graph-jmp-table.png)
_Figure (3): An excerpt from IDA Pro._

---

## 6. Arrays

```c
int b[5] = {123, 87, 487, 7, 978};

void main() {
    int i;
    int a[5];
    for(i = 0; i < 5; i++) {
        a[i] = i;
        b[i] = i;
    }
}
```

**x86 Assembly:**

```assembly
00401021  mov edx, [ebp+var_18]
00401024  mov [ebp+ecx*4+var_14], edx ; [1] Local Array
00401028  mov eax, [ebp+var_18]
0040102B  mov ecx, [ebp+var_18]
0040102E  mov dword_40A000[ecx*4], eax ; [2] Global Array
```

The memory address of an array depends on its declaration (global or local). It is always accompanied by a register acting as an index multiplied by the size of its elements. (For example: for an integer array the index is multiplied by 4, expressed as `[ecx * 4]`.) Note that array indices start at `0`, so the last element is at index `n-1`.

---

## 7. Structs & Linked Lists

### Struct

```c
struct my_structure {                ; [1]
    int x[5];
    char y;
    double z;
};

struct my_structure *gms;            ; [2]

void main() {
    gms = (struct my_structure *) malloc(sizeof(struct my_structure));
    test(gms);
}
```

**x86 Assembly:**

```assembly
00401053  push 20h                   ; Struct size (32 bytes)
00401055  call malloc
0040105A  add esp, 4
0040105D  mov dword_40EA30, eax      ; Store base address
00401062  mov eax, dword_40EA30
00401067  push eax                   ; [1] Pass pointer to function
00401068  call sub_401000
```

A `struct` is declared as a variable, and values are written into it according to the variables it contains and according to the function call. When declaring the `struct` and allocating space for it with `malloc`, we are provided with the base address, which marks the beginning of the struct.

### Linked List

```c
struct node {
    int x;
    struct node * next;
};
typedef struct node pnode;

void main() {
    pnode * curr, * head;
    int i;
    head = NULL;
    
    for(i=1; i<=10; i++) {           ; [1] Build the nodes
        curr = (pnode *)malloc(sizeof(pnode));
        curr->x = i;
        curr->next = head;
        head = curr;
    }
    
    curr = head;
    while(curr) {                    ; [2] Traverse the nodes
        printf("%d\n", curr->x);
        curr = curr->next;
    }
}
```

**x86 Assembly:**

```assembly
0040107E  mov [esp+18h+var_18], 8    ; Node size
00401085  call malloc
0040108A  mov [ebp+var_4], eax       ; eax holds the new node address
0040108D  mov edx, [ebp+var_4]
00401090  mov eax, [ebp+var_C]
00401093  mov [edx], eax             ; [1] curr->x = i
00401095  mov edx, [ebp+var_4]
00401098  mov eax, [ebp+var_8]
0040109B  mov [edx+4], eax           ; [2] curr->next = head
0040109E  mov eax, [ebp+var_4]
004010A1  mov [ebp+var_8], eax       ; head = curr
```

The fundamental difference between a conventional struct and a linked list in assembly is the profusion of `mov` operations that indicate the construction of the links between nodes, as shown in the lines marked `[1]` and `[2]`.
