---
title: "Cyber Attack Lifecycle"
description: "Tracing the typical attack path and understanding the attacker's mindset."
date: 2026-03-20T11:00:00+03:00
slug: "cyber-attack-lifecycle"
weight: 5
hex: "0x5"
stage: "deep-dive"
categories: [deep-dive, malware-analysis]
tags: [reverse-engineering, malware-analysis, malware-reverse-engineering]
translationKey: "attack-flow"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah

When engaging with malicious software (malware) that warrants examination, one must understand the mindset of the adversary and the objectives they pursue. Therefore, in this research we examine the attack lifecycle and explain several fundamental concepts in malware development.

The attack cycle is divided into three principal stages:

![Attack Lifcycle](/assets/img/workshop/lifecycle/lifecycle.png)
_Figure (1): Illustration of the attack lifecycle._

---

## Stage 1: Initial Access

This stage focuses on breaching the initial defensive perimeter and gaining a foothold within the target system.

- **Reconnaissance:** The operation begins with the collection of information about the target. This encompasses identifying IP addresses, scanning for open ports, and even gathering intelligence on personnel for use in social engineering.
- **Exploitation:** Upon identifying a vulnerability (a flaw in the system, an unpatched service, or even a deceivable employee), the threat actor employs a specific tool or code to exploit that weakness.
- **Infiltration:** This is the moment of successful exploitation and traversal of the security boundary, whereby the adversary obtains an initial foothold within the network.

---

## Stage 2: Entrenchment & Discovery

Once the adversary has gained entry, they must understand their new environment and ensure they are not evicted from it.

- **Internal Reconnaissance:** The adversary is now inside the network and begins to survey their surroundings to determine which assets are available, what privileges they hold, and where sensitive data resides (such as databases or administrative servers).
- **Entrenchment:** This step is critical; the adversary does not wish to lose access should the system administrator reboot the server or patch the initial vulnerability. Consequently, they establish backdoors or implant malware to ensure persistence.
> Note: This is precisely the point at which reverse engineering and malware code analysis become critically important for understanding how the adversary conceals themselves and maintains their privileges within the system.

---

## Stage 3: Objective Execution

Here the adversary begins to reap the benefits of the compromise and execute the operation for which they initiated the intrusion.

- **Command & Control (C2):** The implanted malware establishes communication with external servers under the adversary's control. Through this channel, the adversary continuously and covertly issues and receives commands to and from the compromised system.
- **Exfiltration:** The aggregation of sensitive data (credit card numbers, trade secrets, passwords) and its covert transfer beyond the victim's network boundary.
- **Purge:** The final step, consisting of clearing logs and deleting the tooling the adversary employed to conceal any evidence of their presence. In certain cases, this stage may encompass destruction of the system (such as encrypting files in ransomware attacks or wiping them via wiper malware).

---

> In summary: We now possess an understanding of the methodology underlying every attack conducted over the internet. As reverse engineers and malware analysts, our work concentrates primarily on Stage 2 and Stage 3.

---

## Malware Terminology

The preceding steps involve specific technical terminology that we must understand in order to facilitate the analysis process:

### 1. Compression / Packing

This involves combining compressed files via a packer together with decompression code into a single executable. The concept is that, upon reaching the victim's machine, the file is in effect a wrapper program whose primary function is to decompress the genuine malicious payload and execute it in memory in order to evade antivirus products.

![Compression](/assets/img/workshop/lifecycle/compression.png)
_Figure (2): Illustration of how files are compressed and decompressed._

### 2. Obfuscation

A deliberate act of producing code that is difficult for humans (and analysts) to comprehend or read with ease.

- Simple strings appear encrypted with algorithms such as Base64 or XOR.
- Non-functional functions are inserted to distract attention (junk code).
- In assembly language, one may observe an abundance of NOP (No Operation) instructions that perform no function.
- The repeated use of `push` instructions as a technique to conceal string construction in memory.

![Obfuscation](/assets/img/workshop/lifecycle/obf.png)
_Figure (3): Code illustrating how obfuscation is performed._

The following is an illustrative example of how to design obfuscated code in C:

![C obfucation](/assets/img/workshop/lifecycle/c-obf.gif)
_Figure (4): Illustration of how code is manipulated._

### 3. Persistence

The malware developer seeks to ensure that the malware executes on the host and persists for as long as possible, even following a reboot.

- **Special Files:** The program is placed in hidden system paths or those that ordinary users rarely inspect, such as the `%APPDATA%` folder. Certain of these paths afford deeper access privileges within the system.
- **Advanced Techniques:** Further research may be conducted on the use of shared files, access via namespaces, or the use of Alternative Data Streams (ADS) within the NTFS file system.

### 4. Privilege Escalation

The exploitation of a flaw in the system's design or configuration to obtain elevated access to system resources with administrator (Admin) or root privileges.

Common techniques:

- DLL Hijacking
- DLL Injection
- Buffer Overflow
- Stack Overflow
- Heap Spray
- ROP (Return-Oriented Programming)
- UAC Bypasses

### 5. Defense Evasion

This refers to techniques used to evade detection by security controls (such as antivirus and EDR) in order to arouse less suspicion.

Common techniques:

- Killing AV
- Deleting itself after run
- Time bombs / Time stomping
- DLL Side-Loading
- Process Hollowing
- Code Injection
