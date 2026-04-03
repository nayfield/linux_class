# Linux Native Developer Curriculum

A self-contained curriculum for learning Linux native development in C, from the ground up. No prior programming or Linux experience is assumed.

## How to Use This Curriculum

Work through the units in order — each builds on the previous one. Read the concepts, study the code examples, and complete the exercises before moving on. Return to earlier units when a later one references them.

## Units

| # | Title | Topics |
|---|-------|--------|
| [01](units/01_memory_fundamentals.md) | Memory Fundamentals | CPU caches, DRAM, virtual memory |
| [02](units/02_linux_environment.md) | The Linux Environment | Shell, filesystem, man pages, CLI tools |
| [03](units/03_art_of_command_line.md) | The Art of the Command Line | Shell shortcuts, scripting, text processing, one-liners |
| [04](units/04_git_and_version_control.md) | Git & Version Control | Commits, branches, rebase, bisect, collaboration |
| [05](units/05_c_fundamentals.md) | C Language Fundamentals | Types, control flow, functions, strings |
| [06](units/06_compilation_and_build.md) | Compilation & Build Tools | gcc, Makefiles, cmake |
| [07](units/07_pointers_and_memory.md) | Pointers & Memory Management | malloc/free, stack vs heap, memory bugs |
| [08](units/08_data_structures_in_c.md) | Data Structures in C | Linked lists, hash tables, BST, arenas |
| [09](units/09_filesystem_and_io.md) | Filesystem & File I/O | POSIX file API, file descriptors |
| [10](units/10_mmap_and_advanced_io.md) | mmap & Advanced I/O | Memory-mapped files, sendfile, io_uring |
| [11](units/11_processes_and_signals.md) | Processes & Signals | fork/exec/wait, signals, /proc |
| [12](units/12_ipc.md) | Inter-Process Communication | Pipes, FIFOs, Unix sockets, shared memory, message queues |
| [13](units/13_threads_and_sync.md) | Threads & Synchronization | pthreads, mutexes, condition variables |
| [14](units/14_debugging_and_profiling.md) | Debugging & Profiling | gdb, valgrind, strace, perf, sanitizers |
| [15](units/15_networking_and_sockets.md) | Networking & Sockets | BSD socket API, epoll, TCP/IP |
| [16](units/16_libraries_and_linking.md) | Libraries & Dynamic Linking | .a/.so files, dlopen, pkg-config |
| [17](units/17_testing_in_c.md) | Testing in C | Unity framework, TDD, gcov, test doubles |
| [18](units/18_packaging_and_deployment.md) | Packaging & Deployment | systemd, static binaries, Docker |

## Required Reference Material

- **Ulrich Drepper — "What Every Programmer Should Know About Memory"**
  Local copy: [cpumemory.pdf](cpumemory.pdf)
  Read before Unit 01 and again after Unit 07.

## Books and References

Resources mentioned throughout the curriculum, collected here for convenience.

### Books

| Title | Author(s) | Units |
|-------|-----------|-------|
| *What Every Programmer Should Know About Memory* | Ulrich Drepper | 01, 08 |
| *The C Programming Language* (K&R) | Brian Kernighan & Dennis Ritchie | 05, 07, 08 |
| *C Programming: A Modern Approach* | K.N. King | 05 |
| *The Practice of Programming* | Brian Kernighan & Rob Pike | 08 |
| *Advanced Programming in the UNIX Environment* (APUE) | W. Richard Stevens & Stephen Rago | 09, 11, 12 |
| *Unix Network Programming* | W. Richard Stevens | 18 |
| *Test Driven Development for Embedded C* | James Grenning | 17 |
| *Expert C Programming* | Peter van der Linden | 18 |
| *Systems Performance* | Brendan Gregg | 18 |
| *The Linux Command Line* | William Shotts | 02 |

### Online Resources

| Resource | URL | Units |
|----------|-----|-------|
| The Art of the Command Line | https://github.com/jlevy/the-art-of-command-line | 03 |
| The Missing Semester of Your CS Education | https://missing.csail.mit.edu/ | 03 |
| Pro Git | https://git-scm.com/book/en/v2 | 04 |
| GNU Make Manual | https://www.gnu.org/software/make/manual/ | 06 |
| CMake Documentation | https://cmake.org/cmake/help/latest/ | 06 |
| Lord of the io_uring | https://unixism.net/loti/ | 10 |
| Debugging with GDB (GNU manual) | https://sourceware.org/gdb/current/onlinedocs/gdb/ | 14 |
| Valgrind Manual | https://valgrind.org/docs/manual/ | 14 |
| Beej's Guide to Network Programming | https://beej.us/guide/bgnet/ | 15 |
| Unity Test Framework | https://github.com/ThrowTheSwitch/Unity | 17 |
| CMocka | https://cmocka.org/ | 17 |
| Debian Packaging Guide | https://www.debian.org/doc/manuals/maint-guide/ | 18 |
| Docker Documentation | https://docs.docker.com/ | 18 |
| musl libc | https://musl.libc.org/ | 18 |
| The Linux Command Line (free online) | https://linuxcommand.org/tlcl.php | 02 |

## Prerequisites

- A Linux system (native install or VM — Ubuntu/Debian recommended for beginners)
- A terminal and a text editor
- Curiosity

## Recommended Tools

Install these before starting:

```bash
sudo apt install build-essential gdb valgrind strace linux-perf cmake
```
