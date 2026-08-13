---
title: "Flare-On 2015: Solving Challenge 2"
description: "Solving the second Flare-On (2015) challenge: an extensionless PE file inspected with HxD, then static analysis with IDA Pro to crack a complex verification algorithm (XOR + rotate + rolling sum) and write an IDC script to recover the password."
date: 2026-08-13T09:00:00+03:00
slug: "flare-on15-2"
translationKey: "flare-on15-2"
categories: [reverse-engineering, ctf]
tags: [ctf, flare-on, x86, assembly, ida-pro, pe-format, hxd, idc, crypto]
ShowToc: true
draft: false
---

بسم الله

In this article, we are going to solve the second challenge of **Flare-On (2015)**. As usual, we rely on the **Learning-by-Doing** methodology and the **Just-in-Time Learning** approach.

## 1. Downloading and Extracting the Challenge

We download the challenge from the official website: [flare-on.com](https://flare-on.com/)

> Archive password: **flare**.

After extraction, we get a file **without an extension**, so we inspect it in **HxD** to determine its real type:

![Inspecting the file in HxD](/assets/img/flareon15-2/hxd.png)
_Figure (1): Inspecting the file in HxD_

---

## 2. Identifying the File Type and Renaming It

As shown in the image, the file is a **PE (Portable Executable)**, so we rename it to `.exe`.

---

## 3. First Run and Observing the Behavior

After renaming, we run the program from a CMD window:

![Running the program](/assets/img/flareon15-2/cmd1.png)
_Figure (2): Running the program_

The program shows:

```text
You crushed that last one! Let's up the game.
Enter the password>
```

**Meaning:** "You beat the previous challenge! Let's up the game," then it asks for the password. After typing anything, it gives us the result.

Let's dive into the program and statically analyze it in **IDA Pro**.

---

## 4. Static Analysis: A Weird Prologue

At the start of the code, we notice a **deliberate mistake** in the prologue of `sub_401000`:

```asm
sub_401000 proc near
.text:00401000 pop     eax
.text:00401001 push    ebp
.text:00401002 mov     ebp, esp
```

Normally, a function begins with `push ebp` to build its own stack frame, but here a `pop eax` instruction was added **before it**. This instruction pops the **return address** off the top of the stack — or whatever operand was pushed before the call — into `eax`. We will see its importance shortly.

Looking at the code, we see that **two handles** are defined: one for writing and one for reading:

- **Write handle** `[ebp-8]`: (stdout, message, `0x43`).
- **Read handle** `[ebp-0Ch]`: (stdin, `unk_402159`, `0x32`).

```asm
.text:00401004                 sub     esp, 10h
.text:00401007                 mov     [ebp-10h], eax
.text:0040100A                 push    0FFFFFFF6h        ; STD_INPUT_HANDLE
.text:0040100C                 call    GetStdHandle
.text:00401012                 mov     [ebp-0Ch], eax
.text:00401015                 push    0FFFFFFF5h        ; STD_OUTPUT_HANDLE
.text:00401017                 call    GetStdHandle
.text:0040101D                 mov     [ebp-8], eax
.text:00401020                 push    0
.text:00401022                 lea     eax, [ebp-4]
.text:00401025                 push    eax
.text:00401026                 push    43h               ; 'C'
.text:00401028                 push    offset aYouCrushedThat ; "You crushed that last one! Let's up the game."
.text:0040102D                 push    dword ptr [ebp-8]
.text:00401030                 call    WriteFile
.text:00401036                 push    0
.text:00401038                 lea     eax, [ebp-4]
.text:0040103B                 push    eax
.text:0040103C                 push    32h               ; '2'
.text:0040103E                 push    offset unk_402159
.text:00401043                 push    dword ptr [ebp-0Ch]
.text:00401046                 call    ReadFile
```

---

## 5. Calling the Verification Function

Going further down, we see **six operands** pushed onto the stack, but we only need to focus on **three**:

```asm
.text:0040104C                 push    0               ; lpOverlapped
.text:0040104E                 lea     eax, [ebp-4]
.text:00401051                 push    eax             ; lpNumberOfBytesWritten
.text:00401052                 push    11h             ; nNumberOfBytesToWrite
.text:00401054                 push    dword ptr [ebp-4]      ; number of bytes read
.text:00401057                 push    offset unk_402159      ; user input
.text:0040105C                 push    dword ptr [ebp-10h]    ; secret operand (the pop eax value)
.text:0040105F                 call    sub_401084
.text:00401064                 add     esp, 0Ch
```

The three important operands:

1. `[ebp-10h]`: the value that was first popped by `pop eax` — could be an XOR key, the expected value, or a return address.
2. `unk_402159`: the user input.
3. `[ebp-4]`: the actual length of the input the user typed.

The other three operands stay on the stack and do not affect execution — **they are just distraction**.

After that, `sub_401084` verifies the flag, because right after the call there is a `test` followed by a conditional jump, and the condition is `eax != 0`.

---

## 6. Analyzing the Verification Function sub_401084

Let's dig deeper into the function:

```asm
sub_401084 proc near
    push    ebp
    mov     ebp, esp
    sub     esp, 0          ; no extra space (just a frame)
    push    edi
    push    esi
    xor     ebx, ebx        ; ebx = 0 (rolling sum bx)
    mov     ecx, 25h        ; ecx = 37 (required byte count)
    cmp     [ebp+arg_8], ecx ; if input length < 37 -> fail
    jl      loc_4010D7
    mov     esi, [ebp+arg_4]  ; esi = pointer to the input buffer
    mov     edi, [ebp+arg_0]  ; edi = pointer to the hidden buffer
    lea     edi, [edi+ecx-1]  ; edi = edi + 36 (points to the last hidden byte)
```

Clearly, the function takes **3 arguments** using the **cdecl** calling convention. Going back to the calling code, we identify them:

1. `arg_0 (ebp+8)` = `[ebp-10h]` → the hidden data.
2. `arg_4 (ebp+0Ch)` = `unk_402159` → the user input.
3. `arg_8 (ebp+10h)` = `[ebp-4]` → the input length.

Key points to focus on:

- `ebx = 0`: the rolling sum.
- `ecx = 37`: a counter, most likely the correct flag length.
- `esi`: points to the start of the user input.
- `edi`: points to the end of the hidden array `hidden[36]`.

---

## 7. The Main Loop loc_4010A2

```asm
loc_4010A2:
    mov     dx, bx        ; dx = low part of rolling sum (bx)
    and     dx, 3         ; dx = bx & 3 (take only the last two bits)
    mov     ax, 1C7h      ; ax = 0x1C7
    push    eax           ; save eax on the stack
    sahf                  ; load ah (0x01) into EFLAGS flags -> set CF = 1
    lodsb                 ; al = a byte of user input (from esi), then esi++
    pushf                 ; save the current flags on the stack
    xor     al, [esp+4]   ; al = al XOR 0xC7
    xchg    cl, dl        ; swap cl and dl (cl = counter, dl = bx&3)
    rol     ah, cl        ; rotate ah (0x01) by (bx&3) -> ah = 1 << (bx&3)
    popf                  ; restore the flags (CF = 1)
    adc     al, ah        ; al = al + ah + CF
    xchg    cl, dl        ; restore cl and dl
    xor     edx, edx      ; edx = 0
    and     eax, 0FFh     ; eax = al only
    add     bx, ax        ; bx = bx + al (rolling sum)
    scasb                 ; compare al with [edi] then edi++ (DF=0)
    cmovnz  cx, dx        ; if not equal (ZF=0): cx = dx = 0 -> early fail
    pop     eax           ; restore 0x1C7 from the stack
    jecxz   loc_4010D7    ; if cx == 0 -> exit with failure
    sub     edi, 2        ; edi = edi - 2 (net edi-- after the scasb increment)
    loop    loc_4010A2    ; decrement ecx and jump while ecx != 0
    jmp     short loc_4010D9 ; success
```

Let's break these instructions down carefully:

### 7.1 The Secret Behind `xor al, [esp+4]`

```asm
push    eax          ; stores 0x1C7 on the stack
sahf                ; transfers ah (0x01) into the low byte of EFLAGS
lodsb               ; reads a byte of input into al
pushf               ; pushes the current flags onto the stack
```

Now the stack contains: `[esp]` = flags, `[esp+4]` = the value `0x1C7` (in little-endian: `C7 01 00 00`). So `[esp+4]` = `0xC7`:

```asm
xor al, 0xC7
```

### 7.2 The Rotation Power R

```asm
xchg    cl, dl      ; cl = counter, dl = bx&3  -> after the swap: cl = bx&3
rol     ah, cl      ; ah = 1 << (bx&3)
```

Before these instructions:
- `cl` holds the counter `ecx` (starts at 37 and decreases).
- `dl` holds `bx & 3`.

After `xchg cl, dl`, `cl` becomes `bx & 3`, then `rol ah, cl` rotates `ah` (value `0x01`) by `bx & 3`:

| `bx & 3` | `ah` after rotate |
|----------|-------------------|
| 0 | `0x01` |
| 1 | `0x02` |
| 2 | `0x04` |
| 3 | `0x08` |

So `ah` becomes a **power of two**. Let's call it **R**.

### 7.3 The Full Operation

After `popf` we restore the flags saved before `xchg` and `rol`, and the carry flag `CF = 1` (from `sahf` where `ah = 0x01`). This turns `adc al, ah` into:

```text
al = al + R + 1
```

Since `al` was XORed with `0xC7`, the full operation on the input byte is:

```text
T = ((input_byte XOR 0xC7) + R + 1) mod 256
```

Where `T` is the expected (hidden) byte that the result must be compared against.

> **Important:** The value of `R` depends on `bx`, and `bx` is the rolling sum of previous `al` values (i.e., `T` for every byte successfully compared). This is the key point: to compute the required `input_byte`, we need to know `bx` before processing that byte.

---

## 8. Reversing the Loop

The loop processes the bytes of the hidden array from **last to first** (due to `lea edi, [edi+ecx-1]` then `sub edi, 2` with `scasb` each iteration), while reading the input in forward order. So:

```text
hidden[36]  is compared with  input[0]
hidden[35]  is compared with  input[1]
...
hidden[0]   is compared with  input[36]
```

To reverse the process and get the input in the correct order, we walk from `hidden[36]` down to `hidden[0]`. The reverse equation:

```text
(input_byte XOR 0xC7) = (T - R - 1) mod 256
input_byte = ((T - R - 1) mod 256) XOR 0xC7
```

Steps for each byte starting from `hidden[36]` down to `hidden[0]`:

1. Compute `R = 1 << (bx & 3)`.
2. Compute `input_byte = ((T - R - 1) & 0xFF) XOR 0xC7`.
3. Store the result (it will be reversed for now).
4. Update `bx = (bx + T) & 0xFFFF`.

### Worked Example Step by Step

The hidden data (37 bytes) as we extracted it:

```text
hidden = [
  0xAF, 0xAA, 0xAD, 0xEB, 0xAE, 0xAA, 0xEC, 0xA4, 0xBA, 0xAF, 0xAE, 0xAA,
  0x8A, 0xC0, 0xA7, 0xB0, 0xBC, 0x9A, 0xBA, 0xA5, 0xA5, 0xBA, 0xAF, 0xB8,
  0x9D, 0xB8, 0xF9, 0xAE, 0x9D, 0xAB, 0xB4, 0xBC, 0xB6, 0xB3, 0x90, 0x9A, 0xA8
]
```

**Step 1 (i = 36):** `T = 0xA8`, `bx = 0x0000`

- `bx & 3 = 0` → `R = 1`
- `input_byte = ((0xA8 - 1 - 1) & 0xFF) XOR 0xC7 = 0xA6 XOR 0xC7 = 0x61` → `'a'`
- `bx = 0x0000 + 0xA8 = 0x00A8`

**Step 2 (i = 35):** `T = 0x9A`, `bx = 0x00A8`

- `0xA8 & 3 = 0` → `R = 1`
- `input_byte = ((0x9A - 1 - 1) & 0xFF) XOR 0xC7 = 0x98 XOR 0xC7 = 0x5F` → `'_'`
- `bx = 0x00A8 + 0x9A = 0x0142`

**Step 3 (i = 34):** `T = 0x90`, `bx = 0x0142`

- `0x0142 & 3 = 2` → `R = 4`
- `input_byte = ((0x90 - 4 - 1) & 0xFF) XOR 0xC7 = 0x8B XOR 0xC7 = 0x4C` → `'L'`
- `bx = 0x0142 + 0x90 = 0x01D2`

And so on until all bytes are processed.

---

## 9. Where Does the Hidden Data Come From?

Moving to the `start` function, we see a set of confusing instructions. Most likely these opcodes were deliberately placed, and going a bit further we see a block of data defined in the `.text` section. Let's count **37 bytes** from it — that is the hidden array pulled in before `sub_401000` begins:

![The hidden data in IDA](/assets/img/flareon15-2/ida1.png)
_Figure (4): The hidden data inside the .text section_

---

## 10. Writing an IDC Script to Recover the Flag

Now we can write an **IDC** script (available even in the free version of IDA) to find the flag:

```c
#include <idc.idc>

static main()
{
    auto arg0, i, bx, ref_addr, ref_byte, shift, rot, candidate, out, flag;
    arg0 = 0x4010E4;    // address of the hidden data
    bx = 0;
    flag = "";          // accumulate the result as text

    for (i = 0; i < 37; i++)
    {
        // reference address = arg0 + 36 - i (reverse linear walk)
        ref_addr = arg0 + 36 - i;
        ref_byte = Byte(ref_addr);
        shift = bx & 3;
        rot = 1 << shift;   // 1, 2, 4, 8

        for (candidate = 0; candidate < 256; candidate++)
        {
            out = ((candidate ^ 0xC7) + rot + 1) & 0xFF;   // note the XOR with 0xC7
            if (out == ref_byte)
            {
                flag = flag + sprintf("%c", candidate);   // append the character directly
                bx = (bx + out) & 0xFFFF;
                break;
            }
        }
    }

    Message("Flag: %s\n", flag);
}
```

### How to Run the Code Inside IDA?

Go to **File → Script file...** and select the `.idc` file containing the code above.

---

## 11. Result and Final Verification

When the script runs, the following flag appears:

![Script result](/assets/img/flareon15-2/ida2.png)
_Figure (5): The flag produced by the script_

We copy it and try it:

> **`a_Little_b1t_harder_plez@flare-on.com`**

![Verifying the correct flag](/assets/img/flareon15-2/cmd2.png)
_Figure (6): Verifying the correct flag_

And with this, we have successfully solved the second challenge. To be continued, God willing.
