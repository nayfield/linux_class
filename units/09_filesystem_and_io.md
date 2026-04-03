# Unit 09: Filesystem & File I/O

## Overview

On Linux, almost everything is a file. Disks, terminals, network connections, pipes, and devices are all accessed through the same file abstraction. This unit covers the POSIX file API — the system calls that let you open, read, write, and close files directly, without the buffering layer that `stdio` adds. Understanding file descriptors is foundational for everything that follows: processes (Unit 11), networking (Unit 15), and inter-process communication all build on this model.

## Prerequisites

- Unit 07 (Pointers & Memory Management) — reading/writing requires buffers (pointers)
- Unit 02 (The Linux Environment) — familiarity with the filesystem hierarchy

## Learning Objectives

- Use the POSIX file API: `open`, `read`, `write`, `close`, `lseek`
- Understand file descriptors and their relationship to the process
- Query file metadata with `stat`
- Traverse directories with `opendir`/`readdir`
- Understand the difference between POSIX I/O and stdio (buffered vs unbuffered)
- Work with file permissions, symbolic links, and hard links
- Handle I/O errors correctly

## Reading / Resources

- `man 2 open`, `man 2 read`, `man 2 write`, `man 2 close`, `man 2 lseek`
- `man 2 stat`, `man 2 fstat`
- `man 3 opendir`, `man 3 readdir`
- `man 2 symlink`, `man 2 link`
- Advanced Programming in the UNIX Environment (APUE) by Stevens & Rago — Chapters 3–4

## Concepts

### File Descriptors

A **file descriptor** (FD) is a small non-negative integer that represents an open file, device, socket, or pipe. When you open a file, the kernel returns an FD. You use that FD for all subsequent operations.

Every process starts with three FDs already open:

| FD | Name | Description |
|----|------|-------------|
| 0  | stdin | standard input |
| 1  | stdout | standard output |
| 2  | stderr | standard error |

FDs are inherited by child processes and can be passed between processes. They are the fundamental I/O abstraction of the Unix model.

### Opening and Closing Files

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    perror("open");  // prints error message with errno description
    exit(1);
}

// ... use fd ...

close(fd);
```

**Common flags for `open`:**

| Flag | Meaning |
|------|---------|
| `O_RDONLY` | Open for reading |
| `O_WRONLY` | Open for writing |
| `O_RDWR` | Open for reading and writing |
| `O_CREAT` | Create file if it doesn't exist (requires mode arg) |
| `O_TRUNC` | Truncate to zero length on open |
| `O_APPEND` | Writes always go to end of file |
| `O_EXCL` | With `O_CREAT`, fail if file exists |

Creating a file:
```c
int fd = open("newfile.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
//                                                          ^^^^ permissions
```

The mode `0644` means: owner can read+write, group and others can read.

### Reading and Writing

```c
#include <unistd.h>

// read returns: bytes read, 0 at EOF, -1 on error
ssize_t n = read(fd, buf, count);

// write returns: bytes written, -1 on error
ssize_t n = write(fd, buf, count);
```

**Important:** `read` and `write` may read/write fewer bytes than requested (short reads/writes). This is normal and not an error. Always loop:

```c
ssize_t read_all(int fd, void *buf, size_t count) {
    size_t total = 0;
    while (total < count) {
        ssize_t n = read(fd, (char *)buf + total, count - total);
        if (n == 0) break;       // EOF
        if (n == -1) return -1;  // error
        total += n;
    }
    return total;
}
```

### Seeking

```c
#include <unistd.h>

off_t pos = lseek(fd, offset, whence);
```

| `whence` | Meaning |
|----------|---------|
| `SEEK_SET` | Set position to `offset` bytes from start |
| `SEEK_CUR` | Set position to current position + `offset` |
| `SEEK_END` | Set position to end of file + `offset` |

Get current position: `lseek(fd, 0, SEEK_CUR)`
Get file size: `lseek(fd, 0, SEEK_END)`

Not all file descriptors are seekable — pipes, sockets, and terminals return an error on `lseek`.

### Error Handling with errno

When a system call fails, it returns -1 and sets the global `errno` to indicate the error:

```c
#include <errno.h>
#include <string.h>

int fd = open("nonexistent.txt", O_RDONLY);
if (fd == -1) {
    fprintf(stderr, "open failed: %s\n", strerror(errno));
    // or simply:
    perror("open");  // prints "open: No such file or directory"
}
```

Common errno values: `ENOENT` (no such file), `EACCES` (permission denied), `EEXIST` (file exists), `EINTR` (interrupted by signal — may need to retry).

### File Metadata: stat

```c
#include <sys/stat.h>

struct stat st;
if (stat("file.txt", &st) == -1) {
    perror("stat");
    exit(1);
}

printf("size: %ld bytes\n", st.st_size);
printf("inode: %lu\n", st.st_ino);
printf("permissions: %o\n", st.st_mode & 0777);
printf("links: %lu\n", st.st_nlink);
printf("uid: %u\n", st.st_uid);
printf("last modified: %ld\n", st.st_mtim.tv_sec);
```

Check file type:
```c
if (S_ISREG(st.st_mode))  printf("regular file\n");
if (S_ISDIR(st.st_mode))  printf("directory\n");
if (S_ISLNK(st.st_mode))  printf("symbolic link\n");
```

### Directory Traversal

```c
#include <dirent.h>

DIR *dir = opendir("/tmp");
if (!dir) { perror("opendir"); exit(1); }

struct dirent *entry;
while ((entry = readdir(dir)) != NULL) {
    if (entry->d_name[0] == '.') continue;  // skip . and ..
    printf("%s\n", entry->d_name);
}

closedir(dir);
```

### POSIX I/O vs stdio

| | POSIX (`open`/`read`/`write`) | stdio (`fopen`/`fread`/`fwrite`) |
|--|-------------------------------|----------------------------------|
| Abstraction | File descriptors (int) | FILE* streams |
| Buffering | None (unbuffered) | Buffered by default |
| Scope | Linux/UNIX | Portable (C standard) |
| Use for | System programming, sockets, pipes | Text processing, portability |

stdio's buffering reduces system call overhead for many small reads/writes, but can cause problems in network code or when writing to shared files. POSIX I/O is always used when you need precise control.

### Hard Links and Symbolic Links

```c
link("file.txt", "hardlink.txt");        // create hard link
symlink("file.txt", "symlink.txt");      // create symbolic link
unlink("file.txt");                      // remove a name (and file if last link)
```

A **hard link** is a second directory entry pointing to the same inode (same data). Both names are equal — neither is "the original." Deleting one does not affect the other. Hard links cannot cross filesystems or link to directories.

A **symbolic link** is a file that contains a path. It can cross filesystems and link to directories, but breaks if the target is deleted.

## Exercises

### Exercise 1: Implement `cat`

Write a simplified `cat` in C using only POSIX I/O (`open`, `read`, `write`, `close`):
- With no arguments, copy stdin to stdout
- With one or more filename arguments, print each file in order
- Handle errors gracefully (file not found, permission denied)

```bash
./mycat /etc/hostname
./mycat file1.txt file2.txt
echo "hello" | ./mycat
```

### Exercise 2: Implement `wc`

Write a simplified `wc` that counts lines, words, and bytes in a file:

```bash
./mywc file.txt
# Output: 42 317 1823 file.txt
```

Use a fixed-size read buffer (e.g. 4096 bytes) and process it chunk by chunk.

### Exercise 3: Implement `ls -l`

Write a simplified `ls -l` that lists directory contents with:
- File type and permissions (like `-rwxr-xr-x`)
- Number of hard links
- Owner UID (just print the number if you don't resolve names)
- File size
- Filename

Hint: combine `opendir`/`readdir` with `stat` for each entry. Use `man 2 stat` to understand the `st_mode` field and the `S_IRUSR`, `S_IWUSR`, etc. macros.

### Exercise 4: File Copy with Progress

Write a file copy program that:
- Takes source and destination filenames as arguments
- Copies using a configurable buffer size (e.g. `./fcopy -b 65536 src dst`)
- Prints progress as a percentage to stderr
- Handles the case where source and destination are the same file

### Exercise 5: Tail -f

Implement a simplified `tail -f` that monitors a file and prints new content as it is appended:

```c
// Pseudocode:
// 1. Open the file
// 2. Seek to the end
// 3. Loop:
//    a. Try to read
//    b. If nothing to read, sleep briefly (usleep) and try again
//    c. Print whatever was read
```

Test with: in one terminal run `./mytail -f /tmp/test.log`, in another run `echo "line" >> /tmp/test.log` repeatedly.

## What Comes Next

Unit 11 covers processes — `fork`, `exec`, `wait`. Processes inherit their parent's file descriptors, which is how shell pipelines (`cmd1 | cmd2`) work. Understanding file descriptors now makes that click into place.
