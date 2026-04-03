# Unit 12: Inter-Process Communication

## Overview

Processes are isolated by design — they cannot access each other's memory. But programs need to cooperate: a web server spawns worker processes, a database has a separate logger, a shell pipeline connects dozens of programs. Linux provides a rich set of IPC mechanisms, each with different trade-offs in performance, complexity, and persistence. This unit covers the full spectrum: anonymous pipes, named pipes (FIFOs), Unix domain sockets, POSIX shared memory, and POSIX message queues. After this unit you will know which tool to reach for and why.

## Prerequisites

- Unit 11 (Processes & Signals) — fork, exec, process model
- Unit 10 (mmap & Advanced I/O) — mmap, shared memory basics
- Unit 09 (Filesystem & File I/O) — file descriptors, read/write

## Learning Objectives

- Use anonymous pipes for parent-child communication
- Create named pipes (FIFOs) for communication between unrelated processes
- Use Unix domain sockets for bidirectional, full-duplex IPC
- Map shared memory between processes with `shm_open` + `mmap`
- Send messages between processes with POSIX message queues
- Choose the right IPC mechanism for a given use case

## Reading / Resources

- `man 2 pipe`, `man 7 pipe`
- `man 3 mkfifo`, `man 7 fifo`
- `man 2 socketpair`, `man 7 unix`
- `man 3 shm_open`, `man 7 shm_overview`
- `man 3 mq_open`, `man 7 mq_overview`
- APUE by Stevens — Chapters 15–17

## Concepts

### IPC Mechanism Overview

| Mechanism | Direction | Persistence | Best for |
|-----------|-----------|-------------|----------|
| Anonymous pipe | Unidirectional | Process lifetime | Parent-child |
| FIFO (named pipe) | Unidirectional | Until deleted | Unrelated processes, simple |
| Unix socket | Bidirectional | Until deleted | Full-duplex, high-throughput |
| Shared memory | Shared access | Until deleted/reboot | High-speed bulk data |
| Message queue | Unidirectional | Until deleted | Structured messages, priority |
| Signals | Notification only | — | Async notification |

### Anonymous Pipes

A pipe is a one-way byte stream. Data written to the write end can be read from the read end. Pipes have an in-kernel buffer (typically 64KB).

```c
#include <unistd.h>

int fds[2];
pipe(fds);      // fds[0] = read end, fds[1] = write end

pid_t pid = fork();
if (pid == 0) {
    // Child: write to pipe
    close(fds[0]);                          // close unused read end
    write(fds[1], "hello from child\n", 17);
    close(fds[1]);
    exit(0);
} else {
    // Parent: read from pipe
    close(fds[1]);                          // close unused write end
    char buf[64];
    ssize_t n = read(fds[0], buf, sizeof(buf));
    buf[n] = '\0';
    printf("parent got: %s", buf);
    close(fds[0]);
    wait(NULL);
}
```

**Critical rule**: always close the unused end of the pipe in each process. If the write end is not closed in the reader's process, `read` will never return EOF.

Shell pipelines (`cmd1 | cmd2`) are implemented with exactly this pattern: `pipe()` + two `fork()`s + `dup2` to redirect stdout/stdin.

### Implementing a Shell Pipeline

```c
int fds[2];
pipe(fds);

// Fork writer (ls)
pid_t pid1 = fork();
if (pid1 == 0) {
    dup2(fds[1], STDOUT_FILENO);  // redirect stdout to pipe write end
    close(fds[0]);
    close(fds[1]);
    execlp("ls", "ls", "-la", NULL);
    exit(1);
}

// Fork reader (grep)
pid_t pid2 = fork();
if (pid2 == 0) {
    dup2(fds[0], STDIN_FILENO);   // redirect stdin from pipe read end
    close(fds[0]);
    close(fds[1]);
    execlp("grep", "grep", "^-", NULL);
    exit(1);
}

close(fds[0]);
close(fds[1]);
waitpid(pid1, NULL, 0);
waitpid(pid2, NULL, 0);
```

### Named Pipes (FIFOs)

A FIFO is a pipe that has a name in the filesystem. Any process can open it by path — no parent-child relationship required.

```c
#include <sys/stat.h>

// Create the FIFO (like a special file)
mkfifo("/tmp/myfifo", 0666);

// In process A (writer)
int fd = open("/tmp/myfifo", O_WRONLY);  // blocks until a reader opens
write(fd, "message", 7);
close(fd);

// In process B (reader) — run concurrently with A
int fd = open("/tmp/myfifo", O_RDONLY);  // blocks until a writer opens
char buf[64];
read(fd, buf, sizeof(buf));
close(fd);
```

`open` on a FIFO blocks until both ends are open (default behavior). Use `O_NONBLOCK` to open without blocking. Remove with `unlink("/tmp/myfifo")`.

FIFOs are simple and useful for log pipelines: a program writes logs to a FIFO, a consumer reads and processes them.

### Unix Domain Sockets

Unix sockets provide bidirectional, full-duplex IPC with a familiar socket API. They are significantly faster than TCP localhost sockets (no network stack).

```c
#include <sys/socket.h>
#include <sys/un.h>

#define SOCK_PATH "/tmp/myservice.sock"

// Server
int srv = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = {0};
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path) - 1);
unlink(SOCK_PATH);  // remove stale socket
bind(srv, (struct sockaddr *)&addr, sizeof(addr));
listen(srv, 5);

int client = accept(srv, NULL, NULL);
// send/recv on client fd like a TCP socket
```

```c
// Client
int sock = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = {0};
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, SOCK_PATH, sizeof(addr.sun_path) - 1);
connect(sock, (struct sockaddr *)&addr, sizeof(addr));
// send/recv
```

`SOCK_DGRAM` on Unix sockets gives connectionless datagram semantics (like UDP but local).

**Bonus**: Unix sockets support passing open file descriptors between processes using `sendmsg`/`recvmsg` with `SCM_RIGHTS` — the only way to pass an open fd to an unrelated process.

### socketpair

For parent-child communication, `socketpair` creates a connected pair of Unix sockets:

```c
int fds[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, fds);

// Unlike a pipe: both ends can read AND write (full duplex)
// Parent uses fds[0], child uses fds[1]
```

This is how OpenSSH, Chrome, and many other programs implement their worker process communication — cleaner than pipes for bidirectional protocols.

### POSIX Shared Memory

`shm_open` creates a named shared memory object backed by `tmpfs`. Any process that knows the name can map it:

```c
#include <sys/mman.h>
#include <fcntl.h>

// Create and size the shared memory object
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, sizeof(int));

// Map it
int *counter = mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE,
                    MAP_SHARED, fd, 0);
close(fd);  // fd no longer needed after mmap

// Use it
*counter = 42;
printf("shared value: %d\n", *counter);

munmap(counter, sizeof(int));
shm_unlink("/myshm");  // remove (like unlink for files)
```

Link with `-lrt`. Another process opens the same `/myshm` name and maps the same physical memory.

Shared memory is the fastest IPC — no kernel involvement after mapping. But it requires external synchronization (mutexes, semaphores, or atomics) to prevent races. For inter-process mutexes, use `pthread_mutex_t` with `PTHREAD_PROCESS_SHARED` attribute.

### POSIX Message Queues

Message queues send discrete messages with priority, not a byte stream:

```c
#include <mqueue.h>

// Create a queue (name must start with /)
struct mq_attr attr = {
    .mq_maxmsg  = 10,    // max 10 messages in queue
    .mq_msgsize = 256,   // max 256 bytes per message
};
mqd_t mq = mq_open("/myqueue", O_CREAT | O_RDWR, 0666, &attr);

// Send
char msg[] = "hello";
mq_send(mq, msg, strlen(msg), 0 /* priority */);

// Receive (blocks if empty)
char buf[256];
unsigned int priority;
ssize_t n = mq_receive(mq, buf, sizeof(buf), &priority);
buf[n] = '\0';
printf("received (prio %u): %s\n", priority, buf);

mq_close(mq);
mq_unlink("/myqueue");
```

Link with `-lrt`. Higher priority messages are received first. `mq_notify` registers an async notification when a message arrives.

### Choosing IPC: Decision Guide

1. **Parent-child, one direction**: anonymous pipe
2. **Parent-child, two directions**: `socketpair`
3. **Unrelated processes, simple stream**: FIFO
4. **Unrelated processes, full-duplex**: Unix socket
5. **Bulk data, maximum speed**: shared memory + synchronization primitive
6. **Structured messages with priority**: message queue
7. **Simple async notification**: signal (`SIGUSR1`/`SIGUSR2`)
8. **Passing file descriptors**: Unix socket with `SCM_RIGHTS`

## Exercises

### Exercise 1: Multi-Stage Pipeline

Implement `ls | sort | uniq -c | sort -rn | head -5` from scratch in C using anonymous pipes and `fork`/`exec`. Create N pipes for N-1 stages, fork N child processes, and wire up stdin/stdout with `dup2`. Parent waits for all children.

### Exercise 2: Chat with FIFOs

Build a simple two-process chat application using two FIFOs:
- `/tmp/chat_a_to_b` and `/tmp/chat_b_to_a`
- Process A reads from stdin, writes to `a_to_b`, reads from `b_to_a` and prints
- Process B does the reverse
- Both processes run simultaneously using `select` or threads to monitor both FIFOs at once

### Exercise 3: Worker Pool with socketpair

A process manages a pool of N worker processes. The main process:
1. Creates N workers, each with a `socketpair` for communication
2. Distributes work items (integers to compute) to workers via the socket
3. Workers compute (e.g., `isprime(n)`) and send the result back
4. Main collects results and prints them

Demonstrate that multiple workers run in parallel. Verify the `socketpair` is fully bidirectional.

### Exercise 4: Shared Memory Ring Buffer

Implement a producer-consumer ring buffer in shared memory:
- Producer and consumer are separate processes (use `shm_open` + `mmap`)
- Ring buffer holds 64 slots of 64 bytes each
- Use a `pthread_mutex_t` and `pthread_cond_t` with `PTHREAD_PROCESS_SHARED` for synchronization
- Producer pushes 10,000 messages, consumer receives and counts them
- Both processes exit cleanly when done

### Exercise 5: File Descriptor Passing

Use Unix sockets with `SCM_RIGHTS` to pass an open file descriptor from one process to another:
1. Process A opens a file (or a listening socket)
2. Process A connects to a Unix socket server (process B)
3. Process A sends the fd using `sendmsg` with `cmsg` of type `SCM_RIGHTS`
4. Process B receives the fd and uses it directly (reads the file, or accepts connections)

This demonstrates that file descriptors can be transferred between completely unrelated processes — a key primitive for capability-based security and privilege separation.

## What Comes Next

Unit 13 covers threads — lightweight concurrency within a single process. Where IPC crosses process boundaries, threads share the same address space. Many of the synchronization primitives (mutex, condition variable) apply to both, but the threat model and overhead are very different.
