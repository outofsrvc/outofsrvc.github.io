---
title: "Dynamic Lab"
description: "Observing a program while it executes, and defeating its anti-debugging checks."
date: 2026-03-25T10:00:00+03:00
slug: "dynamic-lab"
weight: 10
hex: "0xA"
stage: "practice"
categories: [practice, reverse-engineering]
tags: [dynamic-analysis, x64dbg, anti-debugging, x86-assembly, python]
translationKey: "dynamic"
ShowToc: true
TocOpen: false
draft: false
lab: "/assets/binaries/dyn4m1c_cr4ckm3.7z"
labPassword: "p01nt"
---

Bismillah

In this article, we will solve a simple lab challenge that relies entirely on dynamic analysis using the x64dbg debugger. We assess that the exercise demonstrates how to bypass anti-debugging mechanisms and how to patch the program's execution flow so that it accepts any key we provide.

> Lab environment setup: The executable for this lab can be downloaded from the following link [dyn4m1c_cr4ckm3](/assets/binaries/dyn4m1c_cr4ckm3.7z) in the course repository. Archive password: **p01nt**

---

## 1. Reconnaissance

To begin, we open a command prompt (CMD) and run the program to observe its initial output.

![Program running](/assets/img/workshop/dynamic/dynamic-run.png)
_Figure (1)_

We observe that the program requires an argument (key) to operate.

When inspecting the strings section using the PE-Bear tool, we note an interesting message indicating debugger detection.

![PE-Bear1](/assets/img/workshop/dynamic/PE-bear1.png)
_Figure (2)_

This clearly indicates that a protection routine exists within the code that checks whether the program is running under a debugging environment. By examining the import functions (Imports), we identify the following function:

![PE-Bear2](/assets/img/workshop/dynamic/PE-bear2.png)
_Figure (3)_

The `IsDebuggerPresent` function is a Windows API responsible for detecting whether the program is running inside a debugger. We treat this information as our first pivot point.

---

## 2. Debugging Environment Setup (x64dbg Setup)

We now load the program into **x64dbg**. Since the program requires an argument to be passed at launch, we must inform the debugger accordingly:
1. From the top menu, click `File` -> `Change Command Line`.
2. Append a test word after the program path, for example `test_key`.

![x64dbg1](/assets/img/workshop/dynamic/x1.png)
_Figure (4)_

3. Click `OK`, then press the shortcut `Ctrl + F2` to restart the program and apply the command line.

We then press `F9` (Run) to let the program execute the initial system routines until it halts at the program's main entry point (Entry Point), where the program name will appear in the comments.

![x64dbg2](/assets/img/workshop/dynamic/x2.png)
_Figure (5)_

---

## 3. Bypassing Anti-Debugging

We now seek to locate the point where the program checks for the presence of a debugger in order to neutralize it.

1. Right-click inside the CPU window (code view).

![x64dbg3](/assets/img/workshop/dynamic/x3.png)
_Figure (6)_

2. Select: `Search for` -> `Current Module` -> `String References`.

3. In the strings window that appears, search for the `Debugger detected` message and double-click it to navigate to its location in the code.

![x64dbg4](/assets/img/workshop/dynamic/x4.png)
_Figure (7)_

If we examine the instructions preceding this message, we observe a call to the `IsDebuggerPresent` function, immediately followed by a `test` of the result to determine whether we are operating under a debugger, and subsequently a conditional jump (`je` - Jump if Equal).

**Patching Operation:**
We will now manipulate this conditional jump to invert the logic entirely.
* Single-click on the `je` instruction, then press the `Space` (spacebar) key to open the Edit Instruction window.

![x64dbg5](/assets/img/workshop/dynamic/x5.png)
_Figure (8)_

* Change the instruction from `je` (jump if equal) to `jne` (jump if not equal).

![x64dbg6](/assets/img/workshop/dynamic/x6.png)
_Figure (9)_

We confirm the appearance of the message `Instruction encoded successfully` at the bottom, indicating that the modification was successfully applied in memory. With this, we have effectively blinded the program, and it will no longer detect that we are analyzing it!

---

## 4. Analysis of the Validation Routine and Final Patching (The Core Logic)

We return to the strings window (String References) once again. This time, we search for the success message and navigate to it.

![x64dbg7](/assets/img/workshop/dynamic/x7.png)
_Figure (10)_

Examining the code preceding the success message, we observe a call to the `strcmp` function (the well-known routine that compares two strings: the original key and the key we supplied).

![x64dbg8](/assets/img/workshop/dynamic/x8.png)
_Figure (11)_

We click on the `call` instruction for `strcmp` and press `F2` to set a breakpoint at it (the address will turn red).

We now press `F9` (Run) one or more times until program execution reaches this breakpoint.
**Note: The program will successfully bypass the debugger check thanks to our earlier modification, which can be observed in the CMD window accompanying x64dbg.**

![x64dbg9](/assets/img/workshop/dynamic/x9.png)
_Figure (12)_

### Leaking the Key
Immediately before the `strcmp` call is executed, if we examine the registers window or the stack, we will see that two values were passed to the function:
1. The test key we entered (`test_key`).
2. **The correct original key stored in memory!**

![x64dbg10](/assets/img/workshop/dynamic/x10.png)
_Figure (13)_

### Modifying the Jump Logic (Forcing Success)
We now press `F8` (Step Over) to execute and skip the comparison function. Immediately after the comparison function, we find a `test eax, eax` instruction followed by a conditional jump `jne` (which would jump to the failure message because our key is incorrect).

![x64dbg11](/assets/img/workshop/dynamic/x11.png)
_Figure (14)_

We perform another patch here: click on `jne` and press `Space`, then change it to `je`. We press `F8` to continue. We observe that the Zero Flag (`ZF`) equals zero, and because we changed the instruction to `je`, the program will not take the failure jump; instead it falls through to the success message.

![x64dbg12](/assets/img/workshop/dynamic/x12.png)
_Figure (15)_

We continue pressing `F8` until we reach the `memset` call or the end of the procedure.

![x64dbg13](/assets/img/workshop/dynamic/x13.png)
_Figure (16)_

If we take a look at the CMD window now:

![x64dbg14](/assets/img/workshop/dynamic/x14.png)
_Figure (17)_

> **We have now successfully applied the concept of patching, because the program displayed the "success" message despite the fact that the key we entered (test_key) was incorrect!**

---

## 5. Confirming the Original Key

If we wish to verify our work without patching, we can take the original key we discovered in memory during the analysis prior to the `strcmp` call, and run the program normally from the CMD while passing this key to it.

![x64dbg15](/assets/img/workshop/dynamic/x15.png)
_Figure (18)_

**Congratulations!** You have just reverse engineered a program, bypassed its anti-debugging protection, modified its program logic, and successfully extracted the secret key.
