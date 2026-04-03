# Unit 11: Processes & Signals

## Overview

A process is a running instance of a program. Linux is built around the process model: the kernel creates, schedules, and destroys processes, and they communicate through well-defined mechanisms. This unit covers the process lifecycle — how processes are created with `fork`, replaced with `exec`, and how parents wait for children to finish. It also covers signals, the asynchronous notification mechanism that lets the OS and other processes interrupt your program. By the end you will be able to write a simple shell.

## Prerequisites

- Unit 09 (Filesystem & File I/O) — file descriptors are inherited across `fork`
- Unit 07 (Pointers & Memory Management) — comfort with pointers and memory

## Learning Objectives

- Use `fork` to create child processes
- Use the `exec` family to replace a process image with a new program
- Use `wait`/`waitpid` to reap child processes and avoid zombies
- Understand the process lifecycle and how `/proc` exposes process state
- Install signal handlers with `sigaction`
- Handle `SIGCHLD`, `SIGTERM`, `SIGINT`, and `SIGUSR1` correctly
- Understand which functions are async-signal-safe

## Reading / Resources

- `man 2 fork`, `man 2 execve`, `man 2 waitpid`
- `man 2 sigaction`, `man 7 signal`
- `man 5 proc`
- APUE by Stevens — Chapters 7–10

## Concepts

### The Process Model

Every process has:
- A **PID** (process ID) — a unique integer
- A **PPID** (parent PID) — who created it
- A copy of its parent's **memory** (code, data, stack, heap)
- A copy of its parent's **file descriptors**
- Its own **signal mask** and pending signals

Inspect a process:
```bash
cat /proc/$$/status    # $$ is current shell PID
cat /proc/$$/maps      # memory layout
ls /proc/$$/fd/        # open file descriptors
```

### fork

`fork` creates an exact copy of the calling process:

```c
#include <unistd.h>

pid_t pid = fork();

if (pid == -1) {
    perror("fork");
    exit(1);
} else if (pid == 0) {
    // Child: fork returns 0
    printf("I am the child, PID = %d\n", getpid());
    exit(0);
} else {
    // Parent: fork returns child's PID
    printf("I am the parent, child PID = %d\n", pid);
}
```

After `fork`, parent and child run independently. They have separate copies of all memory (copy-on-write — the kernel shares pages until one writes). They share open file descriptors.

### exec

`exec` replaces the current process image with a new program. The process keeps its PID, file descriptors, and signal handlers (mostly), but the code and memory are replaced.

```c
#include <unistd.h>

// execve is the fundamental system call
// execvp searches PATH, takes NULL-terminated argv array
char *args[] = {"ls", "-la", "/tmp", NULL};
execvp("ls", args);
// If execvp returns, it failed
perror("execvp");
exit(1);
```

The `fork`+`exec` pattern is how Linux runs programs:

```c
pid_t pid = fork();
if (pid == 0) {
    // child: replace with new program
    char *args[] = {"date", NULL};
    execvp("date", args);
    perror("execvp");
    exit(127);
} else {
    // parent: wait for child
    int status;
    waitpid(pid, &status, 0);
    printf("child exited with status %d\n", WEXITSTATUS(status));
}
```

### waitpid and Zombie Processes

When a child exits, it becomes a **zombie** until the parent calls `wait` or `waitpid` to collect its exit status. Zombies consume a PID and a process table entry. Always reap your children.

```c
#include <sys/wait.h>

int status;
pid_t result = waitpid(pid, &status, 0);  // block until child exits

if (WIFEXITED(status)) {
    printf("exited normally: %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("killed by signal: %d\n", WTERMSIG(status));
}
```

To wait for any child: `waitpid(-1, &status, 0)` or `wait(&status)`.

Non-blocking check: `waitpid(pid, &status, WNOHANG)` — returns 0 if child hasn't exited.

### Environment Variables

```c
#include <stdlib.h>

char *path = getenv("PATH");    // get variable value (or NULL)
setenv("MY_VAR", "value", 1);  // set variable (1 = overwrite)
unsetenv("MY_VAR");             // remove variable
```

Child processes inherit the parent's environment. `exec` can pass a custom environment.

### Signals

A signal is an asynchronous notification sent to a process. The process can:
- Ignore it
- Handle it with a custom handler function
- Let the default action occur (usually terminate or stop)

Common signals:

| Signal | Default | Sent by |
|--------|---------|---------|
| `SIGTERM` (15) | Terminate | `kill PID` |
| `SIGKILL` (9) | Terminate (cannot be caught) | `kill -9 PID` |
| `SIGINT` (2) | Terminate | Ctrl+C |
| `SIGQUIT` (3) | Core dump | Ctrl+\ |
| `SIGSEGV` (11) | Core dump | Invalid memory access |
| `SIGCHLD` (17) | Ignore | Child exits/stops |
| `SIGHUP` (1) | Terminate | Terminal closed |
| `SIGUSR1` (10) | Terminate | User-defined |
| `SIGUSR2` (12) | Terminate | User-defined |
| `SIGALRM` (14) | Terminate | `alarm()` timer |

### sigaction

Always use `sigaction` instead of the older `signal()` function — it has cleaner semantics:

```c
#include <signal.h>

static volatile sig_atomic_t g_running = 1;

void handle_sigint(int signo) {
    (void)signo;
    g_running = 0;  // set flag; main loop checks it
}

int main(void) {
    struct sigaction sa = {0};
    sa.sa_handler = handle_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;  // restart interrupted system calls
    sigaction(SIGINT, &sa, NULL);

    while (g_running) {
        // do work
    }
    printf("Exiting cleanly\n");
    return 0;
}
```

**Critical rule:** Signal handlers must only call **async-signal-safe** functions. Most C library functions (`printf`, `malloc`, etc.) are NOT safe to call from a signal handler. The safe pattern is: set a flag in the handler, check the flag in your main loop.

```c
// Async-signal-safe functions (partial list):
// write(), read(), _exit(), kill(), signal()
// Most others are NOT safe
```

### SIGCHLD and Non-blocking Reaping

When a child exits, the parent receives `SIGCHLD`. Use this to reap children without blocking:

```c
void handle_sigchld(int signo) {
    (void)signo;
    // Loop because multiple children may exit simultaneously
    while (waitpid(-1, NULL, WNOHANG) > 0)
        ;
}

struct sigaction sa = {0};
sa.sa_handler = handle_sigchld;
sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
sigaction(SIGCHLD, &sa, NULL);
```

### Sending Signals

```c
kill(pid, SIGTERM);   // send SIGTERM to a specific PID
kill(0, SIGTERM);     // send to entire process group
raise(SIGUSR1);       // send signal to yourself
```

From the command line:
```bash
kill PID          # send SIGTERM
kill -9 PID       # send SIGKILL
kill -SIGUSR1 PID # send SIGUSR1
```

## Exercises

### Exercise 1: Fork Tree

Write a program that creates a tree of processes:
- The initial process forks 2 children
- Each child forks 2 more children
- Leaf processes print their PID and PPID, then exit
- Each parent waits for both children before exiting

Print the full tree. Verify the parent-child relationships with the printed PIDs.

### Exercise 2: Pipe Between Processes

Use `pipe()` + `fork()` to replicate a shell pipeline `ls | sort`:

```c
#include <unistd.h>

int fds[2];
pipe(fds);  // fds[0] = read end, fds[1] = write end

// Fork: child exec's ls with stdout -> fds[1]
// Parent exec's sort with stdin <- fds[0]
// Both must close the unused end of the pipe
```

Hint: use `dup2(fds[1], STDOUT_FILENO)` to redirect stdout to the pipe write end.

### Exercise 3: Signal Handler

Write a program that:
1. Installs a `SIGINT` handler that sets a flag instead of terminating
2. Runs a loop printing "Working..." every second
3. On first Ctrl+C: prints "Caught SIGINT, will exit after this iteration"
4. On second Ctrl+C (or if the flag is set at loop start): exits cleanly

Use `sleep(1)` and handle `EINTR` (sleep interrupted by signal).

### Exercise 4: Simple Shell

Implement a minimal shell that:
1. Prints a `$ ` prompt
2. Reads a line from stdin
3. Splits it into tokens (program name + arguments)
4. Forks a child and execs the command
5. Waits for the child to finish
6. Handles `exit` as a built-in command
7. Handles Ctrl+C (SIGINT) without terminating the shell itself

```bash
$ ls -la
$ echo hello world
$ /bin/date
$ exit
```

Bonus: handle simple `cmd1 | cmd2` pipelines using `pipe()` + two forks.

### Exercise 5: Daemon Process

Write a function `daemonize()` that converts the current process into a daemon:
1. Fork and exit the parent (child becomes orphan, adopted by init)
2. Call `setsid()` to create a new session
3. Change working directory to `/`
4. Redirect stdin/stdout/stderr to `/dev/null`
5. The child then loops, writing a timestamp to a log file every 5 seconds

Verify with `ps aux | grep yourdaemon`.

## What Comes Next

Unit 13 introduces threads — lightweight concurrent execution within a single process. Unlike `fork`, threads share memory, which makes coordination essential and bugs subtle.
