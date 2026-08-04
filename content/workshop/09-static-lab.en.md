---
title: "Static Lab"
description: "Analysing code without ever running it, all the way to recovering the flag."
date: 2026-03-24T10:00:00+03:00
slug: "static-lab"
weight: 9
hex: "0x9"
stage: "practice"
categories: [practice, reverse-engineering]
tags: [static-analysis, ida-pro, xor-encryption, x86-assembly, python]
translationKey: "static"
ShowToc: true
TocOpen: false
draft: false
lab: "/assets/binaries/5t4t1c_cr4ckm3.7z"
labPassword: "p01nt"
---

Bismillah

In this article, we will solve a simple lab challenge that relies entirely on static analysis (Static Analysis). The objective here is to apply the concepts learned previously to decrypt the flag hidden within the program.

> Lab environment setup: You may download the executable for this lab from the following link [5t4t1c_cr4ckm3](/assets/binaries/5t4t1c_cr4ckm3.7z) in the course repository. The archive password is: **p01nt**

---

> To open this file you must open it inside a **VM**. Although I am the one who designed this file, never trust the cyber community. We will run the file with the following command

![Program running](/assets/img/workshop/static/run-static.png)
_Figure (1)_

## 1. Initial Triage

To begin, we open the program using Detect It Easy (DIE) to identify the program's architecture, programming language, and whether it is packed or not.

![DIE](/assets/img/workshop/static/die-static.png)
_Figure (2)_

As seen in the image, the program runs on a 32-bit architecture, was written in the C language, and was compiled using the GCC compiler. The file type is a Windows executable (PE32), and there is no indication of any packer being used.


We then move to PE-Bear to take a quick look at the header sections. After confirming that the program is an EXE, we focus on the Strings and Imports sections with the aim of finding any clear indicator of the Flag.

![PE-Bear](/assets/img/workshop/static/pe-bear.png)
_Figure (3)_

After searching, we discover that the Flag is not present as plaintext, which means it has been concealed or encrypted (Obfuscation).

---

## 2. Analysis Inside IDA Pro

Now we open the executable inside IDA Pro to begin analyzing the code. The program will automatically recognize the file's architecture and disassemble it.

![IDA1](/assets/img/workshop/static/ida1.png)
_Figure (4)_

### Extracting Strings
We begin by searching for any interactive messages that help us locate the verification logic. We press the `Shift + F12` shortcut to open the Strings Window, and search for the phrase that appears upon entering a correct solution (or the error message).

![IDA2](/assets/img/workshop/static/ida2.png)
_Figure (5)_

We double-click on the desired phrase, which takes IDA to the location where this text is stored in the data section (`.rdata` or `.data`).

![IDA3](/assets/img/workshop/static/ida3.png)
_Figure (6)_

### Cross-References
To determine where this text is used in the code, we click on it, then press the `X` shortcut (or double-click on the side comment leading to the function). This takes us directly to the function that calls this message.

We are now in Visual Mode. Recall that you can switch between the graph view (Graph View) and the sequential text view (Text View) by pressing the `Space` button. In our case, we want to remain in `Graph View` to clearly track the program path and the branches (Branches).

---

## 3. Verification Algorithm Analysis

We zoom in on the graph to focus on the basic block that precedes the success message, in an effort to understand the programmatic logic.

![IDA4](/assets/img/workshop/static/ida4.png)
_Figure (7)_

We double-click the function

### A) Length Check
If we focus on the `cmp` comparison instruction, we notice that it is preceded by a call to the `strlen` function (which calculates the length of the entered text). The result is compared against the value `0Bh` (equivalent to the decimal number 11).

![IDA5](/assets/img/workshop/static/ida5.png)
_Figure (8)_

* First conclusion: The correct Flag must consist of 11 characters.

### B) Encryption Loop
We trace the program path to reach the following iteration loop (Loop):

![IDA6](/assets/img/workshop/static/ida6.png)
_Figure (9)_

We clearly observe an `xor` instruction executed using the constant value `5Ah` (or `0x5A`).

![IDA7](/assets/img/workshop/static/ida7.png)
_Figure (10)_

* Second conclusion: The Flag consists of 11 characters, and was encrypted via a simple `XOR` operation using the key `0x5A`.

### C) Extracting the Encrypted Values
To determine the original characters, we must see what is being compared inside the loop.
We observe a `cmp` instruction comparing two registers: `eax` and `edx`.
* The `edx` register holds the characters entered by the user.
* The `eax` register holds the encrypted Flag characters that the program fetches from memory (specifically from the address `byte_407070`).

We single-click on the address `byte_407070` and press the `X` references button.

A references window appears. We inspect the Type column and select the reference carrying the letter **w** (which denotes Write, i.e., the location where these values are written/stored in memory), then press OK.

![IDA8](/assets/img/workshop/static/ida8.png)
_Figure (11)_

The encrypted values stored in memory (as a byte array) now appear, and IDA helpfully displays the accompanying comments. The equation is now complete!

![IDA9](/assets/img/workshop/static/ida9.png)
_Figure (12)_


---

## 4. Decryption Script

We now have an array of 11 encrypted characters, and the encryption key is 0x5A. Since the XOR algorithm is reversible (i.e., encrypting the ciphertext with the same key yields the plaintext), we will write a simple Python script to decrypt and extract the Flag.

```python
# The encrypted values array extracted from IDA
encrypted_hex = [0x29, 0x2e, 0x6a, 0x28, 0x37, 0x1a, 0x29, 0x32, 0x69, 0x36, 0x36]

# The encryption key (XOR Key)
key = 0x5A

# Decrypt via a loop that takes each character (byte), applies XOR with the key, then converts it to text
flag = "".join([chr(b ^ key) for b in encrypted_hex])

print(f"The Flag is: {flag}")
```

When this code is executed, the output will be as follows:

> **The Flag is: st0rm@sh3ll**

-----

## 5. Verification

To confirm the validity of the solution, we run the program and enter the Flag we obtained:

![Flag](/assets/img/workshop/static/the-flag.png)
_Figure (13)_


The success message appears. We have analyzed the file statically, understood the algorithm, and successfully decrypted it!
