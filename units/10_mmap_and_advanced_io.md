# Unit 10: mmap & Advanced I/O

## Overview

The `read`/`write` system calls from Unit 09 work for most programs, but Linux provides more powerful I/O mechanisms that eliminate unnecessary copying, enable shared memory between processes, and support high-throughput asynchronous I/O. This unit covers `mmap` — mapping files and anonymous memory directly into your address space — along with `sendfile`, `splice`, and an introduction to `io_uring`, the modern Linux async I/O interface. Understanding `mmap` also deepens your mental model of virtual memory, tying back to the Drepper paper from Unit 01.

## Prerequisites

- Unit 09 (Filesystem & File I/O) — open, read, write, file descriptors
- Unit 07 (Pointers & Memory Management) — virtual memory layout, pointer arithmetic

## Learning Objectives

- Map files into memory with `mmap` and understand when this is faster than read/write
- Use anonymous `mmap` as an alternative allocator for large buffers
- Share memory between processes using `mmap` on a file or `shm_open`
- Use `msync`, `madvise`, and `mlock` to control mapped memory behavior
- Transfer data between file descriptors without copying with `sendfile` and `splice`
- Understand the `io_uring` model and when to reach for it

## Reading / Resources

- `man 2 mmap`, `man 2 munmap`, `man 2 msync`
- `man 2 madvise`, `man 2 mlock`
- `man 2 sendfile`, `man 2 splice`
- `man 7 shm_overview`, `man 3 shm_open`
- io_uring: https://unixism.net/loti/ (Lord of the io_uring)

## Concepts

### What is mmap?

`mmap` asks the kernel to map a range of a file (or anonymous memory) into your process's virtual address space. You get back a pointer. Reads and writes through that pointer directly access the file — the kernel handles paging data in and out as needed.

```
File on disk:   [page0][page1][page2][page3]
                   ↕       ↕
Virtual memory: [0x...0][0x...1]   (only faulted-in pages are in RAM)
```

The first access to each page causes a **page fault** — the kernel reads the page from disk into RAM. Subsequent accesses hit the page cache and cost nothing extra.

### Basic mmap Usage

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int fd = open("data.bin", O_RDONLY);
if (fd == -1) { perror("open"); exit(1); }

struct stat st;
fstat(fd, &st);
size_t size = st.st_size;

// Map the entire file read-only
void *addr = mmap(NULL,         // let kernel choose address
                  size,         // bytes to map
                  PROT_READ,    // read-only
                  MAP_PRIVATE,  // modifications don't affect the file
                  fd,           // file descriptor
                  0);           // offset into file

if (addr == MAP_FAILED) { perror("mmap"); exit(1); }
close(fd);  // fd can be closed after mmap

// Now access the data directly through the pointer
const char *data = addr;
printf("First byte: %02x\n", (unsigned char)data[0]);

// Unmap when done
munmap(addr, size);
```

### mmap Flags

**Protection flags (`prot`):**
| Flag | Meaning |
|------|---------|
| `PROT_READ` | Pages can be read |
| `PROT_WRITE` | Pages can be written |
| `PROT_EXEC` | Pages can be executed |
| `PROT_NONE` | No access (useful for guard pages) |

**Map flags (`flags`):**
| Flag | Meaning |
|------|---------|
| `MAP_PRIVATE` | Copy-on-write — writes don't modify the file |
| `MAP_SHARED` | Writes go through to the file (and are visible to other mappers) |
| `MAP_ANONYMOUS` | Not backed by a file — zero-initialized memory |
| `MAP_FIXED` | Map at exactly the given address (dangerous) |
| `MAP_POPULATE` | Prefault all pages immediately (no page faults later) |
| `MAP_HUGETLB` | Use huge pages (reduces TLB pressure) |

### Writing to a File with mmap

```c
// Open for read/write, create if needed
int fd = open("output.bin", O_RDWR | O_CREAT | O_TRUNC, 0644);

// Set the file size first (mmap won't extend it)
ftruncate(fd, 4096);

char *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
if (addr == MAP_FAILED) { perror("mmap"); exit(1); }

// Write through the pointer — changes go to the file
memcpy(addr, "Hello, mmap!", 12);

// Flush to disk (optional — kernel will sync eventually)
msync(addr, 4096, MS_SYNC);

munmap(addr, 4096);
close(fd);
```

### Anonymous mmap — Large Allocations

`mmap` with `MAP_ANONYMOUS` creates memory not backed by any file. This is how `malloc` implements large allocations (glibc uses `mmap` for requests > 128KB):

```c
// Allocate 10MB of zeroed memory
size_t size = 10 * 1024 * 1024;
void *buf = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
if (buf == MAP_FAILED) { perror("mmap"); exit(1); }

// Use it...
memset(buf, 0xAB, size);

// Return to OS immediately (unlike free, which may keep it for reuse)
munmap(buf, size);
```

`munmap` returns the memory to the OS immediately, unlike `free` which keeps it in glibc's pool. Use anonymous mmap when you want to allocate and reliably release a large buffer.

### Controlling Mapped Memory

```c
// Tell the kernel about your access pattern (hint, not a command)
madvise(addr, size, MADV_SEQUENTIAL);   // prefetch ahead (streaming reads)
madvise(addr, size, MADV_RANDOM);       // no prefetch (random access)
madvise(addr, size, MADV_WILLNEED);     // start reading these pages now
madvise(addr, size, MADV_DONTNEED);     // drop these pages from RAM (free physical memory without unmapping)

// Lock pages in RAM (prevent them from being swapped out)
mlock(addr, size);                      // requires CAP_IPC_LOCK or small limit
mlockall(MCL_CURRENT | MCL_FUTURE);     // lock entire process address space
```

`MADV_SEQUENTIAL` on a large file read is a significant performance win — the kernel will read-ahead aggressively.

### mmap vs read/write: When to Use Each

| Scenario | Prefer |
|----------|--------|
| Large file, many random accesses | `mmap` |
| File used like a database (random reads) | `mmap` |
| Streaming a file once | `read` with large buffer |
| Writing many small updates | `mmap` + `msync` |
| Network I/O | `read`/`write` (sockets can't be mmapped) |
| Need to handle partial reads carefully | `read`/`write` |
| File > available RAM | `read`/`write` (avoid mapping more than fits) |

mmap avoids one copy (from page cache to user buffer) but adds page fault overhead for initial access. For sequential reads, a well-tuned `read` loop is often comparable or faster.

### sendfile — Zero-Copy File Transfer

`sendfile` transfers data from a file descriptor to another fd (usually a socket) entirely in the kernel — no data is copied to user space:

```c
#include <sys/sendfile.h>

int in_fd  = open("file.bin", O_RDONLY);
// out_fd is typically a socket
off_t offset = 0;
ssize_t sent = sendfile(out_fd, in_fd, &offset, file_size);
```

This is how nginx and lighttpd serve static files efficiently. Without `sendfile`, serving a file requires: `read` (file→user buffer) + `write` (user buffer→socket) = 2 copies. With `sendfile`: 0 copies through user space.

### splice — Kernel Pipeline

`splice` moves data between two file descriptors via a kernel pipe buffer, without any user-space copy:

```c
#include <fcntl.h>

// Copy data from in_fd to out_fd via a pipe
int pipefd[2];
pipe(pipefd);

// Move data from file to pipe (kernel-side)
splice(in_fd, NULL, pipefd[1], NULL, 65536, SPLICE_F_MOVE);

// Move data from pipe to socket (kernel-side)
splice(pipefd[0], NULL, out_fd, NULL, 65536, SPLICE_F_MOVE | SPLICE_F_MORE);
```

`splice` works between any two file descriptors (not just file→socket), making it more flexible than `sendfile`.

### io_uring — Asynchronous I/O

Traditional `read`/`write` are synchronous — they block until the operation completes. `io_uring` (added in Linux 5.1) is a ring-buffer interface for submitting I/O operations asynchronously:

```
Submission Queue (SQE):  your program writes I/O requests here
Completion Queue (CQE):  the kernel writes results here
```

Both are shared memory rings — no syscall needed for each operation. You batch up submissions, call `io_uring_enter` once, and poll the completion queue.

Using the `liburing` wrapper (much simpler than raw syscalls):

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

// Submit an async read
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, sizeof(buf), 0);
sqe->user_data = 42;         // tag for identifying the completion

io_uring_submit(&ring);      // submit all pending SQEs

// Wait for completion
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
printf("read %d bytes (tag=%llu)\n", cqe->res, cqe->user_data);
io_uring_cqe_seen(&ring, cqe);

io_uring_queue_exit(&ring);
```

Install liburing: `apt install liburing-dev`, link with `-luring`.

io_uring excels at: high-IOPS storage workloads, mixed read/write patterns, eliminating per-syscall overhead. It is used by databases (RocksDB, PostgreSQL) and high-performance servers.

## Exercises

### Exercise 1: mmap File Reader

Rewrite the `cat` program from Unit 09 using `mmap` instead of `read`/`write`:
1. `stat` the file to get its size
2. `mmap` it read-only
3. `write` the mapped memory to stdout in one call
4. `munmap`

Compare performance against the `read`-based version on a large file (100MB+):
```bash
dd if=/dev/urandom of=bigfile bs=1M count=100
time ./mycat_read bigfile > /dev/null
time ./mycat_mmap bigfile > /dev/null
```

Use `madvise(MADV_SEQUENTIAL)` on the mmap version and measure again.

### Exercise 2: Memory-Mapped Key-Value Store

Build a simple persistent key-value store backed by a memory-mapped file:
- Format: fixed-size slots of 64 bytes (32-byte key + 32-byte value), with slot 0 as a header containing the count
- Operations: `put(key, value)`, `get(key)`, `delete(key)`
- The store survives process restart (data persists in the file)
- Use `msync(MS_SYNC)` to durably persist after writes

```bash
./kvstore set name Alice
./kvstore get name        # prints Alice
./kvstore set age 30
./kvstore list
# Restart process — data should still be there
```

### Exercise 3: Shared Memory Counter

Two processes increment a shared counter using anonymous mmap:
1. Parent `mmap`s a page with `MAP_SHARED | MAP_ANONYMOUS`
2. Parent forks
3. Both child and parent increment the counter 1,000,000 times
4. Parent waits for child, then prints the counter

Observe the race condition. Then fix it using the atomic operations from `<stdatomic.h>` (`atomic_fetch_add`).

This demonstrates that `MAP_SHARED` memory is literally shared — both processes see the same physical pages.

### Exercise 4: Zero-Copy File Server

Write an HTTP file server that uses `sendfile` to serve static files:
1. Accept TCP connections (from Unit 15 — adapt or write fresh)
2. Parse the `GET /filename HTTP/1.0` request line
3. Open the requested file, `fstat` it for size
4. Send the HTTP headers with `write`
5. Send the file body with `sendfile`
6. Close the connection

Measure throughput with `ab -n 1000 -c 10 http://localhost:8080/bigfile` and compare to a version that uses `read`+`write`.

### Exercise 5: io_uring Batch Read

Using `liburing`, write a program that reads 100 files concurrently using io_uring:
1. Open all 100 files
2. Submit all 100 read SQEs at once
3. Process completions as they arrive
4. Print total bytes read

Compare against reading them sequentially with `read()`. Measure the difference on an SSD.

## What Comes Next

Unit 11 covers processes and signals — `fork`, `exec`, `wait`, and signal handling. The `mmap` skills from this unit reappear in Unit 12 (IPC), where `MAP_SHARED` is used to share memory between unrelated processes.
