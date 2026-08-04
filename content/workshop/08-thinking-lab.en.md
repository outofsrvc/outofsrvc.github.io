---
title: "The Reverse Engineer's Mindset"
description: "Building the reverse engineer's mindset and inferring hidden program logic."
date: 2026-03-23T10:00:00+03:00
slug: "thinking-lab"
weight: 8
hex: "0x8"
stage: "practice"
categories: [practice, reverse-engineering]
tags: [logic-of-re, patching, ascii]
translationKey: "thinking-lab"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah

The reverse engineer does not perceive code merely as complex programming instructions, but rather as a "black box" with inputs and outputs. In this article, we discuss how the reverse engineer's mindset is built and how to infer the hidden programmatic logic.

---

## The Scenario

Imagine we are confronted with a locked electronic door. This door opens only if a single correct character is entered from the keyboard. We have only the keyboard and a small screen that displays a single message upon each input attempt.

### The Experiment
We entered three different characters to observe the door's behavior, and the on-screen results were as follows:

1. Entering A  --> Result: 66
2. Entering B  --> Result: 67
3. Entering C  --> Result: 68

Subsequently, a message appeared indicating that the door opens only when the number 89 is displayed.

---

## The Thought Process

Here the reverse engineer's mind begins by posing three pivotal questions:

1. (Static Analysis): What is the algorithm by which this door operates, based on the inputs and outputs?
2. (Extracting the Flag): What is the precise character that must be entered to reach the required result of opening the door?
3. (Dynamic Analysis & Patching): If we disassemble the keyboard and find a programmatic wire labeled *"If the result is 89, send a signal to open the door"*, and we decide to cut this wire and connect it to a battery to produce a permanent open signal... what do we call this operation?

---

## The Solution and Analysis

### 1. Algorithm Analysis
Upon examining the results, we observe a consistent and steady pattern. In computer science, every character has a numeric value representing it, known as the (ASCII Code).

* The character A has a true value in the computer of 65.
* The character B has a true value of 66.
* The character C has a true value of 67.

Conclusion: The algorithm by which the door operates is (Input + 1). The program takes the entered character's value (ASCII) and adds 1 to it.

### 2. Finding the Flag
The objective is to reach the number 89. Applying the reverse mathematical operation of the algorithm we discovered:
89 - 1 = 88

Consulting the ASCII table, we find that the number 88 represents the character **X**. Therefore, the solution to opening the door through legitimate means is: enter the **X**.

### 3. Patching
The third task explains a fundamental concept in reverse engineering, namely **Patching**.

Rather than searching for the character X (which represents the legitimate password), we modified the program's "behavior" (cut the wires) so that it ignores the check and the comparison operation entirely.

In a real debugging environment (**x64dbg**), this action is exactly analogous to changing the conditional jump instruction **JZ** (jump if the result is equal/zero) to an unconditional jump instruction **JMP** (jump always). This simple modification causes the door to open every time, regardless of whether the entered password is correct or incorrect!
