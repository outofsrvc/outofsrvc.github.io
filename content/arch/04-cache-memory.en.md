---
title: "Cache Memory"
description: "Moving on to memory topics: the cache sitting between CPU and RAM, temporal and spatial locality, direct, fully associative, and set-associative mapping, and cache design elements from capacity to replacement and write policies."
date: 2026-08-25T13:00:00+03:00
slug: "comp-arch-0x4"
translationKey: "comp-arch-0x4"
weight: 5
hex: "0x4"
categories: [computer-architecture]
tags: [cache, memory, sram, dram, locality, mapping-techniques, lru]
ShowToc: true
TocOpen: false
draft: false
---

بسم الله

Having finished the topics concerning the CPU in the last three lessons, we now move on, God willing, to memory topics — which are no less important than what came before them.

Memory topics divide into three areas: (Virtual Memory, Cache Memory, Internal Memory). We have already covered virtual memory from a practical angle in [the virtual memory lesson of the C workshop](../../cworkshop/virtual-memory-stack-vs-heap/); today's topic is **cache memory**.

## Cache Memory Overview

**Cache memory** is a small, high-speed store usually located between the CPU and main memory (RAM). Its job is to hold copies of the most frequently used data and instructions so the processor can reach them quickly.

Cache is typically built from SRAM — much faster than the system's DRAM — which is why it is smaller in capacity and more expensive.

When the processor requests data, the cache is checked first: if the data is present, that is a **cache hit** and access happens very quickly; otherwise it is a **miss**, the slower data is fetched from RAM, and a copy is stored in the cache for future use.

One of the cache's greatest benefits is reducing the Average Memory Access Time by exploiting the principle of Locality of Reference in programs — programs tend to reuse the same data or instructions, or data near locations currently being accessed. When that holds, caching becomes fast and effective.

## Cache Principle

Cache efficiency depends on the hit ratio and miss ratio: for every memory request, if the data is found in the cache it is called a cache hit; otherwise a cache miss.

### The Average Access Time Equation

```text
AMAT = T(hit) + Miss Ratio × T(penalty)
```

where:

- **AMAT:** average memory access time.
- **T(hit):** access time when the data is present in the cache.
- **Miss Ratio:** fraction of accesses not found in the cache.
- **T(penalty):** the extra penalty on a miss — the additional time to fetch the data from main memory.

**A numeric example:** with `T(hit) = 1ns`, penalty `T(penalty) = 100ns`, and a miss ratio of `2%`:

```text
AMAT = 1 + 0.02 × 100 = 3ns
```

Notice how a high hit ratio keeps the average close to the cache's own speed despite slow main memory. Factors influencing the ratio include enlarging the cache, tuning block/line size, and associativity — all aiming to reduce misses and improve performance.

### Quick Test

> Suppose `T(hit)` = 2ns, the penalty `T(penalty)` = 80ns, and the miss ratio is 5% — compute the effective access time yourself before reading on.

<details>
<summary><b>Solution (click to reveal)</b></summary>

```text
AMAT = 2 + 0.05 × 80 = 6ns
```

Notice how a mere 5% miss ratio nearly tripled the time — that is the cache's whole struggle in one line.

</details>

Cache is usually tiered into levels (L1, L2, L3):

![Cache levels L1, L2, L3](/assets/img/computer-arch-org/computer-arch-0x4/cache-hierarchy.png)
_Figure (1): The cache hierarchy_

L1 sits closest to the processor — fastest and smallest — followed by L2, then L3; each higher level is larger but relatively slower. Programs benefit from the cache through temporal and spatial locality: we expect requested data or instructions to be near whatever was just used.

## Locality of Reference

Locality means a program tends to access the same memory region, or nearby regions, within a short period. A quick analogy: think of a delivery courier who prefers delivering to places he has been before, while at the same time picking up orders that are close together so he can deliver them all in one trip.

There are two principal kinds:

**• Temporal Locality:**
Data or instructions accessed recently are likely to be accessed again soon. For example, if a function runs a loop over an array, the processor will likely revisit the same elements.

**• Spatial Locality:**
If a program reaches a particular memory location, it will likely use nearby addresses too. When reading an array sequentially, the current address points to a memory block, and neighboring addresses within that block get cached along with it.

**Example:**

```c
// array accessed sequentially
for (int i = 0; i < N; i++) {
    sum += A[i];
}

// array accessed randomly
for (int i = 0; i < N; i++) {
    int idx = rand() % N;
    sum += A[idx];
}
```

The first loop touches adjacent array elements, exploiting spatial locality and raising the cache-hit ratio. The second loop jumps around randomly via rand, increasing cache misses.

### Quick Test

> A two-dimensional array `int M[1024][1024]` — which loop finishes faster, and why?
>
> ```c
> /* A */                         /* B */
> for (i = 0; i < 1024; i++)      for (j = 0; j < 1024; j++)
>     for (j = 0; j < 1024; j++)      for (i = 0; i < 1024; i++)
>         s += M[i][j];                   s += M[j][i];
> ```

<details>
<summary><b>Solution (click to reveal)</b></summary>

Loop **A** is far faster. It sweeps memory in adjacent order, exploiting spatial locality: every block the cache fetches — typically 64 bytes, i.e. 16 integers — gets fully consumed before moving on. Loop B jumps 4KB at a step, wasting each entire block after touching one element: nearly a miss per access.

</details>

## Mapping Techniques

How is main memory tied to cache? Through a concept called Mapping Techniques:

![Overview of mapping techniques](/assets/img/computer-arch-org/computer-arch-0x4/mapping-overview.png)
_Figure (2): Mapping techniques between main memory and cache_

When moving data from main memory to the cache, one of the following techniques is used:

### Direct Mapping

Each block of main memory maps to exactly one designated slot in the cache (if two blocks sharing the same index are accessed, one evicts the other, causing a conflict).

Here the memory address splits into fields: (Offset, Index, Tag):

![Address fields in direct mapping](/assets/img/computer-arch-org/computer-arch-0x4/direct-fields.png)
_Figure (3): Address split under direct mapping_

- **Index:** selects the candidate cache line.
- **Tag:** if it does not match the stored tag, a cache miss occurs and the current line is replaced.
- **Offset:** an offset locating the byte inside the block:

![The offset's role within a block](/assets/img/computer-arch-org/computer-arch-0x4/direct-offset.png)
_Figure (4): The offset inside a block_

![Direct mapping example](/assets/img/computer-arch-org/computer-arch-0x4/direct-example.png)
_Figure (5): A worked direct-mapping example_

**Advantages:** simple to implement, simple control logic, giving low access time.

**Disadvantages:** conflict probability is high when several blocks share the same index, so the miss rate can rise, and spatial locality support is weak.

### Fully Associative

Any memory block may land in any cache slot because there is no index; the address splits into (Tag, Offset) only:

![Address fields in fully associative mapping](/assets/img/computer-arch-org/computer-arch-0x4/associative-fields.png)
_Figure (6): Address split under fully associative mapping_

Searching for a block compares its tag against all lines until a match is found:

![Fully associative example](/assets/img/computer-arch-org/computer-arch-0x4/associative-example.png)
_Figure (7): A worked fully associative example_

**Advantages:** highly flexible, solves the conflict problem.

**Disadvantages:** expensive hardware (simultaneous comparison against every line).

### Set Associative

This is the middle ground between direct and fully associative: the cache divides into sets, each set holding L lines. One address field selects the set (as in direct mapping), then any line within that set may hold the data:

![Address fields in set associative mapping](/assets/img/computer-arch-org/computer-arch-0x4/setassoc-fields.png)
_Figure (8): Address split under set associative mapping_

![Set associative diagram](/assets/img/computer-arch-org/computer-arch-0x4/setassoc-diagram.png)
_Figure (9): Sets and lines structure_

![Set associative example](/assets/img/computer-arch-org/computer-arch-0x4/setassoc-example.png)
_Figure (10): A worked set associative example_

**Advantages:** combines the speed of direct mapping with the flexibility of full associativity.

**Disadvantages:** costlier and more complex than direct mapping.

## Elements of Cache Design

Cache design is defined by several fundamental parameters: cache size, block/line size, associativity, number of sets, plus the replacement and write policies.

Cache capacity is computed as:

```text
Cache Size = Sets × Block Size × Associativity
```

And the memory address splits into three parts:

- **Offset:** selects the byte inside the block.
- **Index:** selects the line or set number.
- **Tag:** identifies which block of main memory is meant.

This partitioning enables fast lookup in the cache table.

### Replacement Policy

When a set fills up and a new block must come in, an old block has to be chosen for eviction. The replacement policy determines which block gets removed. The major policies:

**• Least Recently Used (LRU):**
Evicts the block untouched for the longest time. It requires tracking line usage (storing age information), effective but complex in hardware.

**• First-In First-Out (FIFO):**
A queue; evicts whichever block entered first, regardless of how often it is reused.

**• Random:**
Picks a random slot for eviction — notable for how simple and fast it is.

Every replacement policy is a trade-off between hit rate and decision time; LRU, for instance, may raise the hit rate but needs extra storage to track ages.

### Write Policies

When writing data to an address already present in the cache, key policies determine how memory is updated:

**• Write-Through:**
Writes the data to both cache and main memory at once. This guarantees synchronized copies and keeps consistency highest — but at a longer write time since the data is written twice.

**• Write-Back:**
Writes the data to the cache first, marks the slot with a dirty bit, and updates main memory later only when that slot is replaced. This speeds up writes but risks data loss if the cache fails before transfer to main memory.

And on a write miss (writing to an address not in the cache), there are two choices:

**• Write-Allocate:**
Load the block from main memory into the cache, then write into it.

**• Write-Around:**
Write directly to main memory without pulling the block into the cache. Used when new data is not expected to be reused soon.

## Hands-On Lab: Measure the Gap Yourself

Write the following program, run it, and compare the two timings:

```c
#include <stdio.h>
#include <time.h>
#include <stdlib.h>

#define N 1000000

int main(void) {
    static int A[N];
    for (long i = 0; i < N; i++) A[i] = i;

    clock_t t = clock();
    long sum = 0;
    for (long i = 0; i < N; i++) sum += A[i];           /* sequential */
    printf("sequential: %ld ms\n", clock() - t);

    t = clock();
    sum = 0;
    for (long i = 0; i < N; i++) sum += A[rand() % N];  /* random */
    printf("random:     %ld ms\n", clock() - t);
    return 0;
}
```

> Run it and record both results, then repeat with N raised to ten million. Does the gap widen or shrink? And think: what happens to the cache once the array outgrows its combined capacity?
>
> **Lab goal:** see locality through your own machine's numbers, not the book's.

---

With this you understand how the cache bridges the gap between a fast processor and slower memory, and how that mediation is managed through mapping, replacement, and write policies. See you in the next lesson, God willing.

## Lesson Summary

| Concept | Key Points |
|---|---|
| Cache | Small ultra-fast SRAM store between CPU and RAM |
| Hit / Miss | A hit means top speed; a miss fetches from RAM and stores a copy |
| AMAT | T(hit) + miss ratio × penalty |
| Temporal locality | Recently used data will likely be used again soon |
| Spatial locality | Fetching a block brings its neighbors along |
| Mapping | Direct (fast/collisions) → set associative (the sweet spot) → fully associative (flexible/costly) |
| Replacement | LRU effective but complex; FIFO simple but blind; Random fastest to decide |
| Writes | Write-through: higher consistency; Write-back: higher speed with a Dirty Bit |

Keep us in your prayers.
