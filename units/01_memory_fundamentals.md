# Unit 01: Memory Fundamentals

## Overview

Before writing a single line of C, you need a mental model of how memory actually works. This unit is a reading assignment: Ulrich Drepper's "What Every Programmer Should Know About Memory." It is a dense technical paper, and you will not understand all of it the first time — that is expected. The goal is to plant questions in your mind that the rest of this curriculum will answer. Every performance decision you make as a Linux developer traces back to the concepts in this paper.

## Prerequisites

None. This is the first unit.

## Learning Objectives

- Understand the difference between CPU registers, cache (L1/L2/L3), and main memory (DRAM)
- Know what a cache line is and why its size matters
- Understand why memory access patterns affect program performance
- Have a basic picture of virtual memory and what the TLB does
- Know that NUMA exists and why it matters on multi-socket systems

## Required Reading

**Ulrich Drepper — "What Every Programmer Should Know About Memory"**
[cpumemory.pdf](../cpumemory.pdf)

Minimum reading for this unit:
- **Section 1** — Introduction
- **Section 2** — Commodity Hardware Today (CPU and DRAM architecture)
- **Section 3** — CPU Caches (the most important section)
- **Section 6** — What Programmers Can Do (practical takeaways)

Sections 4, 5, and 7 are valuable but can be revisited after Unit 07 (Pointers & Memory Management).

## Key Concepts

### The Memory Hierarchy

Modern computers have multiple layers of memory, each faster but smaller than the last:

```
Registers       ~0.3 ns     ~1 KB      (inside the CPU)
L1 Cache        ~1 ns       ~32 KB     (per core)
L2 Cache        ~4 ns       ~256 KB    (per core)
L3 Cache        ~10 ns      ~8 MB      (shared across cores)
Main Memory     ~60 ns      ~GBs       (DRAM)
SSD             ~100 µs     ~TBs
HDD             ~10 ms      ~TBs
```

Your program's performance is largely determined by which level of this hierarchy your data lives in at any moment.

### Cache Lines

The CPU does not fetch individual bytes from memory. It fetches a **cache line** — typically 64 bytes — at a time. This has a profound implication: accessing one byte loads the 63 surrounding bytes into cache for free.

Programs that access memory sequentially (arrays) are fast because each cache miss loads the next 64 bytes. Programs that access memory randomly (pointer-chasing through a linked list) are slow because each access is likely a cache miss.

### Virtual Memory

Every process sees its own private address space — it does not directly access physical RAM addresses. The operating system maintains a **page table** that maps virtual addresses to physical addresses in 4 KB pages. The CPU's **TLB** (Translation Lookaside Buffer) caches recent translations to make this fast.

Understanding virtual memory is essential for understanding `malloc`, `mmap`, and how processes are isolated from each other.

### NUMA (Non-Uniform Memory Access)

On servers with multiple CPU sockets, each socket has its own local memory. Accessing memory attached to a different socket is slower. This is called NUMA. For now, just be aware it exists — it becomes relevant when writing high-performance multi-threaded programs.

## Exercises

### Exercise 1: Reading Journal

Read the required sections of Drepper's paper. As you read, write down:

1. Three things that surprised you.
2. Two questions the paper raised that you do not yet know the answer to.
3. One concept you want to understand better after completing this curriculum.

Keep this list — revisit it after completing Unit 07 and Unit 14.

### Exercise 2: Observe Your Own Machine

Run the following commands on your Linux system and record the output:

```bash
# How many CPU cores and their cache sizes
lscpu | grep -E "^CPU|Cache"

# Detailed cache topology
cat /sys/devices/system/cpu/cpu0/cache/index*/size
cat /sys/devices/system/cpu/cpu0/cache/index*/type
cat /sys/devices/system/cpu/cpu0/cache/index*/coherency_line_size

# How much RAM is available
free -h
```

Match what you see to the hierarchy described in Section 2 of the paper.

### Exercise 3: Intuition Check (no code required)

Given what you now know about cache lines, which of these two loops do you expect to be faster, and why?

**Loop A** — row-major traversal:
```c
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        sum += matrix[i][j];
```

**Loop B** — column-major traversal:
```c
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        sum += matrix[i][j];
```

Write your answer down. You will verify it empirically in Unit 14 (Debugging & Profiling).

## What Comes Next

Unit 02 introduces the Linux environment: the terminal, the shell, and the tools you will use throughout this curriculum. Unit 05 begins C programming. Unit 07 returns to memory with hands-on `malloc`/`free` and the concepts from this paper will start to click into place.
