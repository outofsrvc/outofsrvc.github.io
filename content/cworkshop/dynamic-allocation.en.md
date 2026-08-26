---
title: "Dynamic Memory Allocation"
description: "malloc, calloc, realloc, bins, and tcache — and the misuse vulnerabilities: heap overflow, UAF, and double free."
date: 2026-08-13T10:00:00+03:00
slug: "dynamic-allocation"
weight: 13
hex: "0x12"
stage: "memory"
categories: [c-lang]
tags: ["c-lang", "heap", "memory", "malloc", "exploitation"]
translationKey: "dynamic-allocation"
ShowToc: true
TocOpen: false
draft: false
---

Bismillah.

After discussing the heap and the stack and understanding the internal structure of each, today we cover **dynamic memory allocation** in an expanded way — free-memory management and the most common vulnerabilities in this area.

> **Dynamic Memory Allocation**: understanding it is essential for analyzing software, exploiting vulnerabilities, and understanding how programs work from the inside.

## Core Allocation Functions

### 1. malloc() and free()

**malloc** allocates a heap block of a given size (in bytes) **without initialization** — its contents are garbage values:

```c
void *malloc(size_t size);
```

**free** releases previously allocated memory:

```c
void free(void *ptr);
```

### The Internal Structure of a Chunk (malloc_chunk)

As in the previous article, a chunk in an allocator (like glibc) has this structure:

```c
struct malloc_chunk {
    size_t prev_size;   // previous chunk's size (if it is free)
    size_t size;        // current chunk's size + flags
    struct malloc_chunk *fd;  // pointer to the next free chunk (for free chunks)
    struct malloc_chunk *bk;  // pointer to the previous free chunk
};
```

- **size**: the chunk's total size including the header; the three rightmost bits (LSB) are used for flags.
- **prev_size**: used only if the previous chunk is free (PREV_INUSE = 0), enabling **coalescing** on release to reduce fragmentation.

### How malloc Works (Theory)

1. A chunk of a given size is requested.
2. It searches the bins (free-chunk lists).
3. If no suitable chunk exists, memory is requested from the OS via `sbrk()` or `mmap()`.
4. If the found chunk is larger than needed, it is split and the remainder returned.

### Example

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    // allocate memory for ten integers = 40 bytes
    int *nums = (int*)malloc(10 * sizeof(int));

    // check allocation success
    if (nums == NULL) {
        printf("failed..!\n");
        return 1;
    }

    // use the allocated memory
    for (int i = 0; i < 10; i++) {
        nums[i] = i * 10;
    }

    // release the memory
    free(nums);
    return 0;
}
```

Memory layout:

```text
0x1000: [Flags + chunk size]   -> metadata (hidden from the user)
0x1008: [nums[0]]              -> the nums pointer points here
0x100C: [nums[1]]
  ~ ~ ~
0x1028: [nums[9]]
```

### How free Works (Theory)

1. Validates the pointer (does it point at a chunk's beginning?).
2. If the adjacent chunk is free, it merges them.
3. Returns the chunk to the freed-memory management structures (bins).

> **Focus on `free` — it is where things break.**

### Common Mistakes to Avoid

1. Freeing a pointer you never allocated with an allocation function.
2. Freeing the same chunk twice (Double Free).
3. Using a chunk after freeing it (UAF).
4. After freeing, **set the pointer to NULL immediately**.

## 2. calloc()

Allocates a block for an array of elements **and zeroes it**:

```c
void *calloc(size_t num, size_t size);
```

- Allocates a block fitting `num` elements of `size` each.
- Zero-initializes every byte (the fundamental difference from `malloc`).
- Total computation: `num * size`.

The initialization is done via `memset`:

```c
memset(ptr, 0, num * size);
```

> `memset` fills a memory block with a given value: the pointer, the value to write, and the byte count.

> **Important note**: because of initialization, `calloc(4, 5)` **does not equal** `malloc(20)` — the former has 20 fully-zeroed bytes, the latter garbage. This makes calloc slower but safer.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    // allocate an array of 5 floats
    float *temps = (float*)calloc(5, sizeof(float));

    if (temps == NULL) {
        printf("Failed..!\n");
        return 1;
    }

    // confirm zero-initialization
    for (int i = 0; i < 5; i++) {
        printf("temps[%d] = %f\n", i, temps[i]); // all print 0.0
    }

    free(temps);
    return 0;
}
```

## 3. realloc()

Changes the size of an allocated block — with two scenarios:

1. **In-place expansion**: if the space after the chunk is sufficient (the next chunk is free), the block is expanded in place — and the new pointer equals the old one.
2. **Relocation**: if there is not enough space, a new block is allocated, data is moved via `memcpy()`, the old one freed, and the new pointer returned.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    char *buffer = (char*)malloc(10);

    if (buffer == NULL) return 1;

    strcpy(buffer, "Hello");

    // expand memory to fit a longer string
    char *new_buffer = (char*)realloc(buffer, 20);

    if (new_buffer == NULL) {
        free(buffer); // keep the original block alive
        return 1;
    }

    buffer = new_buffer; // update only after checking
    strcat(buffer, "World"); // now there is enough room

    free(buffer);
    return 0;
}
```

> **The correct pattern**: never write `buffer = realloc(buffer, ...)` directly — if realloc fails you lose the original pointer. Store it in a temporary pointer and check for NULL before updating.

## 4. alloca()

Not expanded here: it allocates on the **stack** and its use is risky (it causes overflow vulnerabilities). A key point: its memory is released automatically when the function exits **without free()** — exactly like the stack.

## Managing Freed Chunks (Bins)

When a chunk is freed with `free()`, it is not returned to the system immediately; it is kept in organized lists called **bins** for later reuse — improving performance and reducing fragmentation.

### The Main Bin Types

| Type | Size | Lists | Pattern | Coalescing |
|---|---|---|---|---|
| **Fast Bins** | small (16-88 byte chunks) | 10 singly-linked lists | LIFO | no (for speed) |
| **Small Bins** | medium (32-1024) | 62 lists | FIFO | yes |
| **Large Bins** | large (>1024) | 63 lists | sorted by size (best-fit) | yes |
| **Unsorted Bin** | — | one temporary list | a staging area before sorting | — |

- **Fast bins**: fixed sizes (16, 24, 32, 40, 48, 56, 64, 72, 80, 88 bytes on x64), LIFO, and no merging of adjacent chunks — they are merged later via `malloc_consolidate()` when full or when a large chunk is requested.
- **Small bins**: FIFO lists; free neighbors merge into larger chunks.
- **Large bins**: sorted by size using best-fit to reduce fragmentation — slower searches.

### The Freed Chunk Lifecycle

When `free(ptr)` is called:

1. The chunk's size is checked: if **≤ 88** it goes to the fast bins; otherwise it goes to the unsorted bin and gets sorted later (small/large).
2. On `malloc()` the search order (since glibc 2.26) is:
   **tcache ← fastbins ← unsorted bins ← small/large bins**.

### A Practical Example

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    void *a = malloc(32);     // small chunk → fast bin
    void *b = malloc(1024);   // large chunk → unsorted bin
    void *c = malloc(2048);   // another large chunk

    free(a);   // goes to a fast bin
    free(b);   // goes to the unsorted bin
    free(c);   // goes to the unsorted bin

    malloc(40); // if a suitable size exists, it comes from the fast bin
    return 0;
}
```

> You can watch this clearly using **GDB with pwndbg** (`heap bins` and `heap chunks`) — an indispensable tool for understanding allocator behavior.

## tcache — Thread-Local Cache

In **glibc 2.26**, **tcache** was introduced to speed up small allocations in multi-threaded environments.

**The problem it solved**: in multi-threaded environments, accessing shared memory (the main arena) caused **contention** on the mutex, slowing performance. So a **per-thread cache** was introduced, managing its own freed chunks without a lock — cutting allocation time by 70-80%.

### tcache Properties

- 64 lists per thread (regardless of architecture).
- Largest chunk: **1032** bytes on x64.
- Each list holds at most **7** chunks.

**On allocation**: if the size qualifies (≤ 1032), it searches the matching tcache list — if a chunk is found it is removed (LIFO) and returned; otherwise the traditional bins are used.

**On free**: if the list is not full (< 7), the chunk goes into tcache; otherwise it is sent to the fast bins or unsorted bins.

## The Most Common Misuse Vulnerabilities

### 1. Heap Overflow

A buffer overflow occurring in the heap region — writing data beyond the allocated size. It causes:

**A. Corruption of adjacent data:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    char *buffer1 = (char*)malloc(10);
    char *buffer2 = (char*)malloc(10);

    strcpy(buffer2, "SECRET");

    strcpy(buffer1, "HEAP-OVERFLOW-ATTACK"); // overflowing buffer1

    printf("buffer2 = %s\n", buffer2); // corrupted by adjacency
    free(buffer1);
    free(buffer2);
    return 0;
}
```

> **Warning**: the actual result depends on the real chunk layout — in modern glibc the metadata may be corrupted before user data, and the corruption may be detected at `free`. Do not rely on the theoretical output literally.

**B. Metadata modification:**

```c
char *chunk1 = (char*)malloc(16);
char *chunk2 = (char*)malloc(16);

memset(chunk1, 'A', 32); // overflow: 16 < 32

free(chunk2); // the system detects metadata damage and may abort
```

**C. Function Pointer Overwrite:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef void (*func_ptr)();

void normal_func() { printf("normal\n"); }
void hacked_func() { printf("hackin\n"); }

int main() {
    func_ptr *fptr = (func_ptr*)malloc(sizeof(func_ptr));
    char *buffer = (char*)malloc(16);

    *fptr = normal_func;

    // overflow buffer to overwrite fptr
    strcpy(buffer, "AAAAAAAAAAAAAAAA"  // 16 characters
                   "\x40\x10\x40");    // hacked_func's address

    (*fptr)(); // hacked_func gets called
    return 0;
}
```

> **A realistic caveat**: `hacked_func`'s address is hardcoded here for simplicity — in practice you need to compile with `-no-pie` and disable ASLR for addresses to be fixed like this.

### 2. Use After Free (UAF)

Occurs when a program uses a pointer to memory **after it has been freed** with `free()`:

**A. Data corruption:**

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *ptr1 = (int*)malloc(sizeof(int));
    *ptr1 = 100;
    printf("the basic value: %d\n", *ptr1); // 100

    free(ptr1);

    // a new chunk of the same size
    int *ptr2 = (int*)malloc(sizeof(int));
    *ptr2 = 200;

    // using the freed pointer (UAF)
    *ptr1 = 300; // writing to freed memory

    printf("the new ptr2: %d\n", *ptr2); // an unexpected value
    return 0;
}
```

**B. Malicious code execution (the most dangerous part):**

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    void (*printFunc)(); // function pointer
} Object;

void legitFunc() { printf("legit function\n"); }
void evilFunc()  { printf("exploit function\n"); }

int main() {
    Object *obj = (Object*)malloc(sizeof(Object));
    obj->printFunc = legitFunc;
    obj->printFunc(); // the normal call

    free(obj); // free the pointer

    // allocate a same-sized chunk under attacker control
    unsigned long *fake = (unsigned long*)malloc(sizeof(Object));
    *fake = (unsigned long)evilFunc;

    obj->printFunc(); // UAF → calls evilFunc
    return 0;
}
```

> **RE connection**: this scenario is precisely the basis of **tcache poisoning** — the easiest and most common heap attack. Reallocating a freed chunk and planting a malicious address in it is exactly what these attacks do.

### 3. Double Free

Freeing the same pointer twice without reallocating in between:

**A. Heap corruption:**

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *ptr = (int*)malloc(sizeof(int));
    *ptr = 10;

    free(ptr); // first free — correct

    // ... instructions ...

    free(ptr); // second free — error: double free
    return 0;
}
```

**B. Malicious code execution:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct Data {
    char buffer[32];
    void (*security_check)();
};

void normal_check()   { printf("normal_check\n"); }
void malicious_code() { printf("hackin code\n"); }

int main() {
    struct Data *d1 = (struct Data*)malloc(sizeof(struct Data));
    d1->security_check = normal_check;

    free(d1); // initial free

    // the attacker controls the freed chunk
    struct Data *d2 = (struct Data*)malloc(sizeof(struct Data));
    struct Data *d3 = (struct Data*)malloc(sizeof(struct Data));

    memset(d2->buffer, 'A', 32);
    d2->security_check = malicious_code;

    free(d1); // double free — corrupts the metadata

    struct Data *d4 = (struct Data*)malloc(sizeof(struct Data));
    d4->security_check(); // malicious code executes
    return 0;
}
```

> **RE connection**: this example is the basis of the **Fastbin Dup Attack** — creating a cycle in the fastbin list via double free to allocate memory at a location you control.

## Why Does This Matter?

> Consider that **45% of vulnerabilities between 2020-2023** were related to memory management issues (per MITRE/Microsoft reports).

### Preventive Measures

**1. Null out pointers after freeing:**

```c
free(ptr);
ptr = NULL; // prevent use-after-free
```

**2. Dynamic verification tools:**

```bash
valgrind --leak-check=full ./programName
```

**3. OS-level protections:**
- **ASLR**: address space layout randomization.
- **DEP/NX**: preventing code execution from non-executable regions.

## The Series' RE Connection

This article is the **complete foundation for heap exploitation**:

1. **Knowing your bins determines your attack**: chunks in tcache mean its attacks are easiest, while fastbins demand understanding their fine differences. When an analyst examines a sample, knowing the heap state tells them what is possible.
2. **UAF → tcache poisoning**, **Double Free → fastbin dup**: every vulnerability explained here has a specific exploitation technique built on it.
3. **Malware analysis**: much malware uses heap tricks to hide code or evade protection — understanding this material makes those techniques readable in the disassembly.

---

God willing, we have covered most matters of dynamic allocation. Some things remain unexplained (like struct and typedef in depth) and are coming in future articles. If I got something right, it is from Allah; whatever is wrong is from myself.













