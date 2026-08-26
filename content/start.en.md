---
title: "Start Here"
description: "The complete roadmap: five interconnected sections taking you from \"how does a processor work?\" to disassembling a real program yourself."
slug: "start"
translationKey: "start"
ShowToc: false
---

بسم الله — Peace be upon you.

This blog is not a collection of scattered articles to read in any order; it is **one progressive curriculum** that takes you from the question "how does a processor work?" all the way to disassembling a real program yourself. This page is the roadmap.

With how heavily AI is used these days, explaining how to use it properly to students became a must — so our journey begins with the article [How to Use AI](/ar/posts/how-to-use-ai/) *(Arabic)*.

The series were designed as one connected whole: each series prepares you for the next. I tried my best to weave practical examples into the Computer Architecture and Operating Systems series — since they are purely theoretical — so the student never gets bored studying them.

What I want to make clear: these series do not replace reading the original books behind them, but God willing they will be of great benefit, getting you familiar with roughly ninety percent of the terminology. Each series has a primary reference it was built upon. The series have also been translated into English using AI — so if you spot any mistakes, we ask your forbearance.

> **Before starting the C, Assembly, and Entry-P01NT series:** you need to set up a virtual machine with Windows 10, and install the gcc compiler, IDA, and the tools mentioned in the Entry-P01NT workshop.

## The Full Path

| # | Series | Lessons | Status |
|---|---|---|---|
| 1 | Computer Architecture | 8 | Available |
| 2 | Operating Systems | 7 | Available |
| 3 | C Workshop | 19 | Available |
| 4 | Assembly | 7 + 3 lab solutions | Available |
| 5 | Entry-P01NT Workshop | 11 + 2 labs | Available |

## 1 · Computer Architecture (Arch-P01NTs)

The foundation for everything after it: how a computer is organized inside — the processor and its structure, the instruction set (ISA), addressing modes, cache and main memory, the I/O system, and interconnects. You will write C code and see its footprint in assembly from the very first lesson.

→ [Start with lesson 0x0 — Computer Architecture & Organization](../../en/arch/comp-arch-0x0/)

## 2 · Operating Systems (OS-P01NTs)

The layer your programs live in: system structure and syscalls, interrupts, processes, threads, and scheduling, virtual memory management, and the file-system interface. Without them, much of what you see in a disassembler stays unexplained magic.

→ [Start with lesson 0x0 — The Basic Structure of Operating Systems](../../en/os/os-0x0/)

## 3 · C Workshop (C-P01NTs)

The language closest to the machine a reverse engineer analyzes daily, taught here through the RE lens: every concept tied to its footprint in the binary — compilation stages, pointers and memory, stack versus heap, dynamic allocation and its exploits, ending at the preprocessor.

→ [Start with lesson 0x00 — The Stages Code Goes Through](../../en/cworkshop/compilation-stages/)

## 4 · Assembly (Asm-P01NTs)

The practical bridge between C and reverse engineering, written live inside IDA Pro: registers, stack frames, the cdecl/stdcall calling conventions, string instructions and REP prefixes, jump tables, and struct rebuilding — plus real labs with detailed solutions. It is a practical prerequisite before entering the workshop.

→ [Start with lesson 0x1 — x86 Basics](/ar/asm/x86-basics/) *(Arabic — English version coming soon)*

## 5 · Entry-P01NT Workshop (Reverse Engineering)

The destination: the reverse engineer's mindset, Windows architecture and the PE format, recognizing C constructs inside assembly, the cyber-attack lifecycle, static and dynamic analysis tools — ending with two hands-on labs you work through yourself.

→ [Start with lesson 0x1 — Introduction to Reverse Engineering](../../en/workshop/introduction-to-re/)

## Three Rules of Study

1. **Labs run inside an isolated VM only** — no exceptions, no matter how innocent the sample looks.
2. **Reproduce everything yourself**: write the code, run it, analyze it by hand — reading alone is an illusion of learning.
3. **Ask for hints, not solutions**: wrestling with the problem is the learning itself; anything that feeds you ready-made answers steals that struggle from you.
