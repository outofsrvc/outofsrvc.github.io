---
title: "Introduction to Reverse Engineering"
description: "An entry point into reverse engineering and why it matters."
date: 2026-03-16T10:00:00+03:00
slug: "introduction-to-re"
weight: 1
hex: "0x1"
stage: "foundations"
categories: [foundations, intro]
tags: [reverse-engineering, malware-analysis, basics]
translationKey: "introduction-to-re"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah

## Introduction

From childhood, human beings possess an innate curiosity to understand *"how things work?"*. It is this curiosity that drives a child to break apart a favorite toy in order to see the small motor inside it. In the advanced world of technology, this curiosity evolves into a precise and critical engineering discipline known as reverse engineering.

Most computer users may not have heard of reverse engineering before, perhaps because it is not as widespread as the field of hacking, or perhaps because they know it by another term, namely "cracking".

Many differ in their definition of reverse engineering, but in short we can state:

> Reverse engineering: the process of analyzing something in order to understand its mechanism of operation.

Reverse engineering is therefore divided into two main branches:
1. Software reverse engineering: Reverse Code Engineering (RCE)
2. Hardware reverse engineering: Hardware Reverse Engineering (HRE)

In this workshop, we will focus on and delve into RCE.

---

## Why is There a Need for Reverse Engineering in the First Place?

This varies depending on the reverse engineer and their objectives:

### 1. The Programmer's Perspective (Developer)

If you are the author of the program, you will most likely wish to debug your software in order to discover errors or their root causes. After discovering and correcting errors, you may wish to assess the strength of the program's protection, that is, its susceptibility to being defeated by crackers. In this case, you would reverse engineer your own program with the aim of hardening it against cracking.

### 2. The Cracker's Perspective

Crackers' motivations for learning reverse engineering differ from person to person or team to team:
* Challenge: adopting it as a tool to defeat protections and obfuscation techniques.
* Knowledge for all: sharing with others the techniques they have discovered.
* Breaking monopolies: assisting users in obtaining expensive software.
* Sabotage: adopting it as a tool for disruption, whether for financial motives (such as defeating a competitor's software) or for other reasons.

---

## 🚁 To Simplify the Concept: The Drone Scenario

No example is clearer than the world of weapons for simplifying this complex concept.

Imagine the following scenario: a highly advanced reconnaissance drone has crashed relatively intact within your territory. This aircraft is a "black box" to you; you observe the end result (an aircraft that flies, spies, and evades detection), yet you possess neither the engineering schematics, nor the source code that drives its engines, nor its encrypted communications system.

![Drone as a black box](/assets/img/workshop/intro/drone-blackbox.jpg)
_Figure (1): Treating unknown technology as a black box._

In this scenario, the objective of reverse engineering is not to destroy the aircraft, but to understand it so that you can:
1. Build defensive counter-systems against it.
2. Replicate the technology for your own advantage.
3. Identify weaknesses in order to disable it in the future.

---

## What is Reverse Engineering Precisely?

Reverse engineering is the process of analyzing a system (mechanical, electronic, or software) in order to determine its components and the relationships among them, and using this analysis to recreate a representation of the system that illustrates how it works (such as schematics or source code).

In the software context (the most prevalent today), you begin with a ready-to-run executable file (.exe or ELF) and lack the source code (Source Code) written by the programmers (such as C/C++).
Your objective is: to transform this complex binary file (Machine Code) into a human-readable language in order to understand the detailed operation of the code (Functionality).

---

## ⚙️ Stages of Reverse Engineering

Whether the target is a mechanical weapon such as a rifle, or an electronic one such as malicious software, the process passes through four fundamental stages:

### 1. Reconnaissance

Before touching anything, the target's behavior is observed:
* In weapons: how are they loaded? What is the rate of fire? What type of ammunition is used?
* In software: the program is executed and its behavior is monitored (its network connections, the files it creates, and how it consumes memory).

### 2. Disassembly

* In weapons: disassembling the rifle piece by piece, screw by screw, to understand the mechanical mechanism of the trigger and firing chamber.
* In software: using tools called disassemblers to convert machine code (0s and 1s) into Assembly Language, a low-level language that accurately describes the operations executed by the processor.

### 3. Analysis and Comprehension

Here the arduous mental work begins, where the disassembled pieces are connected to understand the overall logic:
* In weapons: understanding that the particular shape of the feeding component is what enables automatic firing.
![Hardware RE: mechanism analysis](/assets/img/workshop/intro/hardware-re.jpg)
_Figure (2): Analysis of the mechanical mechanism of hardware and solid components._

* In software: attempting to trace the execution path (Control Flow) through the code in order to reach the core "algorithm", such as locating the encryption algorithm.
![Software RE: Code analysis](/assets/img/workshop/intro/software-re.jpg)
_Figure (3): Analysis of code flow and programming logic._

### 4. Reconstruction (Decompilation)

Reaching the highest level of understanding, where Assembly language is converted into a high-level programming language (such as C) using decompilers, thereby allowing the programming logic to be understood in a form closer to human language.

---

## Conclusion

Reverse engineering is not merely knowledge of how to use tools such as disassemblers and debuggers, but rather a mindset. It demands enormous patience, the ability to solve complex puzzles, and a very deep knowledge of how computers, operating systems, and processors work.

> By understanding how to "take apart" technology, the reverse engineer acquires the ability to build better and more secure technology, much as the maker of armor learns by studying "projectiles".
