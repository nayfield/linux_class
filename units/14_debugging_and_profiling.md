# Unit 14: Debugging & Profiling

## Overview

Professional Linux developers spend as much time debugging as writing code. This unit covers the full debugging toolkit: gdb for interactive debugging, valgrind for memory error detection, strace/ltrace for tracing system calls and library calls, and perf for CPU performance profiling. It also introduces the compiler sanitizers — AddressSanitizer and ThreadSanitizer — which catch entire classes of bugs at much lower overhead than valgrind. By the end you will be able to diagnose and fix bugs that would otherwise take days to find.

## Prerequisites

- All previous units — debugging requires understanding the full stack
- Unit 06 (Compilation & Build Tools) — you need `-g` flag and understand object files

## Learning Objectives

- Use gdb to set breakpoints, step through code, inspect variables, and read backtraces
- Analyze core dumps with gdb
- Use valgrind memcheck to find all memory bugs from Unit 07
- Use valgrind massif to profile heap usage
- Use strace to trace system calls of any running program
- Use ltrace to trace library calls
- Profile CPU hotspots with perf
- Use AddressSanitizer and ThreadSanitizer in your regular build workflow

## Reading / Resources

- `man 1 gdb`
- `man 1 valgrind`
- `man 1 strace`
- `man 1 perf`
- Debugging with GDB (GNU manual): https://sourceware.org/gdb/current/onlinedocs/gdb/
- Valgrind manual: https://valgrind.org/docs/manual/

## Concepts

### gdb — The GNU Debugger

Always compile with `-g` to include debug symbols:

```bash
gcc -Wall -g -O0 program.c -o program
gdb ./program
```

**Essential gdb commands:**

```gdb
(gdb) run                    # start the program
(gdb) run arg1 arg2          # start with arguments
(gdb) break main             # breakpoint at function
(gdb) break file.c:42        # breakpoint at line
(gdb) break *0x400abc        # breakpoint at address
(gdb) info breakpoints       # list breakpoints
(gdb) delete 2               # delete breakpoint #2

(gdb) continue               # continue until next breakpoint
(gdb) next                   # step over (don't enter functions)
(gdb) step                   # step into function
(gdb) finish                 # run until current function returns
(gdb) until 55               # run until line 55

(gdb) print x                # print variable value
(gdb) print *ptr             # dereference and print pointer
(gdb) print arr[0]@10        # print 10 elements of array
(gdb) display x              # print x automatically at each stop
(gdb) info locals            # show all local variables
(gdb) info args              # show function arguments

(gdb) backtrace              # show call stack (bt for short)
(gdb) frame 2                # switch to stack frame #2
(gdb) up / down              # move between frames

(gdb) watch x                # watchpoint: stop when x changes
(gdb) watch *ptr             # watchpoint on memory address

(gdb) list                   # show source code
(gdb) layout src             # TUI mode — source code view
(gdb) quit                   # exit gdb
```

**Attaching to a running process:**
```bash
gdb -p PID
```

### Core Dumps

When a program crashes (SIGSEGV, SIGABRT, etc.), it can write a core dump — a snapshot of memory at the time of the crash.

Enable core dumps:
```bash
ulimit -c unlimited        # allow unlimited core dump size
# Run the crashing program
gdb ./program core         # analyze the dump
```

In gdb, `backtrace` immediately shows where the crash occurred.

Configure core dump file location:
```bash
echo '/tmp/core.%e.%p' | sudo tee /proc/sys/kernel/core_pattern
```

### Valgrind Memcheck

Valgrind runs your program in a sandboxed environment that detects every invalid memory access:

```bash
gcc -Wall -g program.c -o program
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./program
```

Key output:
```
==1234== Invalid write of size 4
==1234==    at 0x4005B2: main (program.c:15)
==1234==  Address 0x5204080 is 0 bytes after a block of size 20 alloc'd
==1234==    at 0x4C2FB0F: malloc (...)
==1234==    by 0x4005A1: main (program.c:12)

==1234== LEAK SUMMARY:
==1234==    definitely lost: 40 bytes in 1 blocks
```

Valgrind slows your program ~10-50x — use it during development and in CI, not production.

### Valgrind Massif (Heap Profiler)

```bash
valgrind --tool=massif ./program
ms_print massif.out.*
```

Massif generates a detailed timeline of heap usage. Use it to find unexpected memory growth.

### strace — System Call Tracer

`strace` shows every system call a program makes:

```bash
strace ./program           # trace from the start
strace -p PID              # attach to running process
strace -e trace=file ./program   # filter to file-related calls only
strace -e trace=network ./program  # filter to network calls
strace -o output.txt ./program     # save to file
strace -c ./program        # summary statistics only
```

Use cases:
- Find out why a program is hanging (often `read` waiting on a pipe or socket)
- Diagnose "permission denied" errors (see which file it's trying to open)
- Understand what a program does without reading the source code

```bash
# Example: see what files a program opens
strace -e openat ls /tmp 2>&1 | grep openat
```

### ltrace — Library Call Tracer

Like strace but for shared library calls:

```bash
ltrace ./program
```

Shows calls to `malloc`, `free`, `printf`, `strcmp`, etc. Useful for understanding library-level behavior.

### perf — Linux Performance Counters

`perf` uses hardware counters to profile CPU usage with minimal overhead:

```bash
# Record a profile
perf record -g ./program

# View the report
perf report

# Quick stats
perf stat ./program

# Count cache misses
perf stat -e cache-misses,cache-references ./program
```

The `perf report` output is a call graph showing which functions consume the most CPU time. Navigate with arrow keys, press `a` to annotate with source lines.

**Flame graphs** (visual representation of perf output):
```bash
perf record -F 99 -g ./program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### AddressSanitizer (ASan)

ASan catches memory bugs at near-native speed (~2x slowdown vs valgrind's 10-50x):

```bash
gcc -Wall -g -fsanitize=address -fno-omit-frame-pointer program.c -o program
./program
```

Detects: buffer overflows (stack and heap), use-after-free, use-after-return, double-free, memory leaks.

ASan output example:
```
==1234==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000054
READ of size 4 at 0x602000000054 thread T0
    #0 0x400abc in main program.c:15
    #1 0x7f...  in __libc_start_main
```

**Use ASan in your regular debug builds** — it catches bugs as soon as you introduce them.

### ThreadSanitizer (TSan)

TSan detects data races at runtime:

```bash
gcc -Wall -g -fsanitize=thread program.c -o program -lpthread
./program
```

TSan output:
```
WARNING: ThreadSanitizer: data race (pid=1234)
  Write of size 4 at 0x... by thread T1:
    #0 increment race.c:8
  Previous read of size 4 at 0x... by main thread:
    #0 main race.c:22
```

Note: TSan and ASan cannot be used together — choose one per build.

### UndefinedBehaviorSanitizer (UBSan)

Catches undefined behavior: integer overflow, null pointer dereference, misaligned access, etc.:

```bash
gcc -Wall -g -fsanitize=undefined program.c -o program
```

### The Debugging Workflow

1. **Reproduce** the bug reliably. If it's intermittent, find the trigger.
2. **Narrow it down**: binary search with `printf`, then switch to gdb.
3. **ASan first**: compile with `-fsanitize=address` and run. It catches ~80% of C bugs instantly.
4. **valgrind** if you suspect memory corruption that ASan missed.
5. **gdb** when you need to understand execution flow and inspect state.
6. **strace** when the bug is at the system call level (file not found, permission errors).
7. **perf** when the program is correct but too slow.

## Exercises

### Exercise 1: Debug a Segfault

This program crashes. Do not read the code carefully first — use gdb to find the bug:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

Node *list_create(int n) {
    Node *head = NULL;
    for (int i = 0; i < n; i++) {
        Node *node = malloc(sizeof(Node));
        node->value = i;
        node->next = head;
        head = node;
    }
    return head;
}

int list_sum(Node *head) {
    int sum = 0;
    while (head->next != NULL) {  // bug here
        sum += head->value;
        head = head->next;
    }
    return sum;
}

int main(void) {
    Node *list = list_create(5);
    printf("sum = %d\n", list_sum(list));
    return 0;
}
```

Steps:
1. Compile with `gcc -Wall -g -o buggy buggy.c`
2. Run under gdb, get the backtrace
3. Identify the bug
4. Fix it

### Exercise 2: Memory Bug Hunt with Valgrind + ASan

Write a program that has all four memory bug types from Unit 07 (leak, use-after-free, double-free, buffer overflow). Then:
1. Find each bug with valgrind
2. Find each bug with ASan
3. Compare the output — which tool is more readable? Which is faster?

### Exercise 3: Cache Miss Profiling

From Unit 01 Exercise 3: implement both row-major and column-major matrix traversal for a large matrix (N=2000). Profile both with:

```bash
perf stat -e cache-misses,cache-references,instructions ./matrix_row
perf stat -e cache-misses,cache-references,instructions ./matrix_col
```

Confirm your Unit 01 prediction. How much larger is the cache-miss rate for the slower version?

### Exercise 4: strace Investigation

Pick any command-line tool you use regularly (e.g. `ls`, `cat`, `grep`). Run it under `strace -c` to see a summary of its system calls. Then run with `-e trace=file` to see all file operations.

Answer:
1. Which system call dominates by count?
2. What files does it open that you didn't expect?
3. Does it access `/proc` or `/sys`?

### Exercise 5: Race Condition Detection

Take the broken counter program from Unit 13 Exercise 1 (without the mutex fix) and run it under ThreadSanitizer. Observe the TSan report. Then add the mutex fix and confirm TSan reports clean.

## What Comes Next

Unit 15 covers networking and sockets — building programs that communicate over the network using the same file descriptor model you learned in Unit 09, with the tools from this unit to diagnose when things go wrong.
