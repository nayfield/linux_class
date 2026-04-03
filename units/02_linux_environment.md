# Unit 02: The Linux Environment

## Overview

Linux is not just an operating system — it is an environment. Before writing programs, you need to be fluent in navigating that environment from the command line. This unit covers the terminal, the shell, the filesystem layout, permissions, and the core CLI tools you will use every day as a Linux developer. If you are already comfortable in a Linux terminal, skim this unit and jump to the exercises.

## Prerequisites

- Unit 01 (Memory Fundamentals) — reading assignment complete

## Learning Objectives

- Navigate the Linux filesystem from the command line
- Understand the standard filesystem hierarchy (/bin, /usr, /etc, /proc, /dev, /sys)
- Use man pages to look up documentation for any command or system call
- Manage files, permissions, and processes from the terminal
- Write basic bash one-liners using pipes and redirection
- Understand what standard input, standard output, and standard error are

## Reading / Resources

- `man hier` — the filesystem hierarchy manual page
- `man bash` — the bash manual (reference, not a cover-to-cover read)
- The Linux Command Line by William Shotts (free online: https://linuxcommand.org/tlcl.php) — Chapters 1–10

## Concepts

### The Terminal and the Shell

A **terminal** is a program that gives you a text interface. The **shell** is the program running inside it that interprets your commands. On most Linux systems the default shell is **bash** (Bourne Again Shell).

When you open a terminal you see a prompt, typically something like:

```
rod@hostname:~$
```

The `~` means your current directory is your home directory (`/home/rod`). The `$` means you are a normal user (not root).

### The Filesystem Hierarchy

Linux organizes everything under a single root `/`. Key directories:

| Path | Contents |
|------|----------|
| `/bin`, `/usr/bin` | User programs (ls, cat, grep, gcc…) |
| `/sbin`, `/usr/sbin` | System administration programs |
| `/etc` | Configuration files |
| `/home` | User home directories |
| `/tmp` | Temporary files (cleared on reboot) |
| `/var` | Variable data: logs, databases, mail |
| `/lib`, `/usr/lib` | Shared libraries (.so files) |
| `/proc` | Virtual filesystem — kernel/process info |
| `/sys` | Virtual filesystem — hardware/driver info |
| `/dev` | Device files (disks, terminals, /dev/null…) |
| `/mnt`, `/media` | Mount points for external storage |

`/proc` and `/sys` are not real files on disk — they are windows into the running kernel. You will use them constantly.

### Essential Commands

**Navigation:**
```bash
pwd           # print working directory
ls -la        # list files (long format, including hidden)
cd /path      # change directory
cd ~          # go home
cd -          # go to previous directory
```

**Files:**
```bash
cp src dst    # copy
mv src dst    # move or rename
rm file       # delete (no trash — gone forever)
mkdir dir     # create directory
touch file    # create empty file or update timestamp
cat file      # print file contents
less file     # page through a file (q to quit)
```

**Searching:**
```bash
grep pattern file      # search for pattern in file
grep -r pattern dir/   # search recursively
find . -name "*.c"     # find files by name
which gcc              # find where a program lives
```

**Processes:**
```bash
ps aux         # list all running processes
top            # live process monitor (q to quit)
kill PID       # send SIGTERM to a process
kill -9 PID    # send SIGKILL (force kill)
```

**Redirection and Pipes:**
```bash
command > file      # redirect stdout to file (overwrite)
command >> file     # redirect stdout to file (append)
command 2> file     # redirect stderr to file
command 2>&1        # redirect stderr to stdout
cmd1 | cmd2         # pipe stdout of cmd1 to stdin of cmd2
```

### File Permissions

Every file has three permission sets: owner, group, others. Each set has read (r=4), write (w=2), execute (x=1).

```bash
ls -l myfile
# -rwxr-x--- 1 rod developers 4096 Jan 1 12:00 myfile
#  ^^^         owner
#     ^^^      group
#        ^^^   others
```

```bash
chmod 755 file     # rwxr-xr-x
chmod +x file      # add execute for all
chown user file    # change owner
```

For executable C programs, you always need the execute bit set.

### Man Pages

Man pages are the authoritative documentation for every command and system call on Linux. Every professional Linux developer reads man pages.

```bash
man ls         # manual for ls command
man 2 open     # section 2 = system calls (open, read, write…)
man 3 printf   # section 3 = C library functions
man 7 signal   # section 7 = overview/concepts
```

Sections that matter for C development:
- **Section 2**: System calls (`man 2 read`, `man 2 fork`)
- **Section 3**: C library functions (`man 3 malloc`, `man 3 printf`)
- **Section 7**: Concepts and overviews (`man 7 pthreads`, `man 7 tcp`)

### Standard Streams

Every process has three standard file descriptors open by default:

| FD | Name | Default |
|----|------|---------|
| 0  | stdin | keyboard |
| 1  | stdout | terminal |
| 2  | stderr | terminal |

In C:
```c
printf("hello\n");           // writes to stdout (fd 1)
fprintf(stderr, "error\n");  // writes to stderr (fd 2)
```

## Exercises

### Exercise 1: Filesystem Exploration

Without using a file manager, answer these questions using only terminal commands:

1. How many files are in `/usr/bin`?
2. What is the size of the `gcc` binary?
3. What does `/proc/cpuinfo` contain?
4. What does `/proc/self/maps` show? (Hint: run `cat /proc/self/maps` and look up `man 5 proc`)
5. What device files exist in `/dev` that relate to your terminal? (Hint: `ls /dev/tty*`)

### Exercise 2: Man Page Practice

Look up and summarize (in one sentence each) the purpose of:

1. `man 2 write`
2. `man 3 fopen`
3. `man 2 mmap`
4. `man 7 signal`

You will use all of these in later units.

### Exercise 3: Pipeline

Using only `ps`, `grep`, `awk`, and `sort`, write a single pipeline that:
1. Lists all running processes
2. Filters to only those owned by your user
3. Prints just the PID and command name
4. Sorts by PID

### Exercise 4: Write a Shell Script

Create a file called `sysinfo.sh` that prints:
- The current date and time
- The hostname
- The kernel version (`uname -r`)
- The amount of free memory
- The top 5 processes by CPU usage

Make it executable and run it.

```bash
chmod +x sysinfo.sh
./sysinfo.sh
```

## What Comes Next

Unit 03 (The Art of the Command Line) deepens your shell skills before you start writing C. Unit 05 begins C programming — you will use the terminal constantly from this point forward.
