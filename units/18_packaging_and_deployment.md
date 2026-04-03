# Unit 18: Packaging & Deployment

## Overview

Writing a working C program is the beginning — getting it into production is the end goal. This unit covers the Linux deployment ecosystem: packaging your software as a `.deb` or `.rpm`, running it as a managed service with systemd, building statically linked binaries for maximum portability, and containerizing with Docker. After this unit you will be able to take a C application from a local build to a properly installed, automatically started system service.

## Prerequisites

- All previous units
- Unit 06 (Compilation & Build Tools) — Makefiles and CMake

## Learning Objectives

- Understand how Linux package managers install software
- Create a minimal `.deb` package for Debian/Ubuntu
- Write a systemd service unit file
- Manage services with `systemctl`
- Build fully static binaries with musl libc
- Write a Dockerfile that builds and runs a C application
- Understand cross-compilation for different architectures

## Reading / Resources

- `man 5 systemd.service`, `man 1 systemctl`
- Debian packaging guide: https://www.debian.org/doc/manuals/maint-guide/
- Docker documentation: https://docs.docker.com/
- musl libc: https://musl.libc.org/

## Concepts

### Linux Package Concepts

A package is a compressed archive containing:
- The program binary (and libraries)
- Configuration files
- Metadata (name, version, description, dependencies)
- Install/remove scripts

Package managers (`apt`, `dnf`, `pacman`) handle dependency resolution, installation, and upgrades.

Two dominant formats:
| Format | Used by | Tool |
|--------|---------|------|
| `.deb` | Debian, Ubuntu, Mint | `dpkg`, `apt` |
| `.rpm` | Fedora, RHEL, CentOS | `rpm`, `dnf` |

### Creating a .deb Package (Minimal)

A `.deb` is a specific archive structure:

```
myprogram_1.0_amd64.deb
├── control.tar.gz     (package metadata)
│   └── control        (name, version, arch, depends, description)
└── data.tar.gz        (files to install, with paths)
    └── usr/bin/myprogram
```

Using the `dh-make` / `dpkg-deb` approach:

```bash
# Create directory tree mirroring the install paths
mkdir -p myprogram_1.0_amd64/usr/bin
mkdir -p myprogram_1.0_amd64/DEBIAN

# Copy the binary
cp myprogram myprogram_1.0_amd64/usr/bin/

# Write the control file
cat > myprogram_1.0_amd64/DEBIAN/control << EOF
Package: myprogram
Version: 1.0
Architecture: amd64
Maintainer: Your Name <you@example.com>
Description: A minimal example program
 This is a longer description of the package.
 Can span multiple lines with a space at the start.
EOF

# Build the .deb
dpkg-deb --build myprogram_1.0_amd64

# Install it
sudo dpkg -i myprogram_1.0_amd64.deb

# Remove it
sudo dpkg -r myprogram
```

For production packages, use `debhelper` and `dh` — they handle the boilerplate.

### systemd Service Units

systemd is the init system and service manager on modern Linux. It starts services at boot, restarts them on failure, and manages their logs.

A service unit file (`/etc/systemd/system/myservice.service`):

```ini
[Unit]
Description=My Example Service
After=network.target

[Service]
Type=simple
User=myservice
ExecStart=/usr/bin/myprogram --config /etc/myprogram/config.toml
Restart=on-failure
RestartSec=5s

# Resource limits
LimitNOFILE=65536
MemoryMax=512M

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/myprogram /var/log/myprogram

[Install]
WantedBy=multi-user.target
```

**Service types:**
- `Type=simple` — ExecStart process is the main service process
- `Type=forking` — ExecStart forks and the parent exits (traditional daemon style)
- `Type=notify` — Service sends `sd_notify(READY=1)` when ready

### Managing Services with systemctl

```bash
# Enable at boot + start now
sudo systemctl enable --now myservice

# Start / stop / restart
sudo systemctl start myservice
sudo systemctl stop myservice
sudo systemctl restart myservice
sudo systemctl reload myservice  # send SIGHUP (if the service supports it)

# Check status
systemctl status myservice

# View logs (live)
journalctl -u myservice -f

# View logs from last boot
journalctl -u myservice -b

# Reload unit files after editing
sudo systemctl daemon-reload
```

### Writing a Proper Daemon in C

A systemd-managed daemon is simpler than a traditional Unix daemon (no need for `fork`, `setsid`, pid files):

```c
#include <signal.h>
#include <stdlib.h>
#include <stdio.h>
#include <syslog.h>

static volatile sig_atomic_t g_running = 1;

void handle_sigterm(int sig) { (void)sig; g_running = 0; }
void handle_sighup(int sig)  { (void)sig; /* reload config */ }

int main(void) {
    openlog("myservice", LOG_PID, LOG_DAEMON);
    syslog(LOG_INFO, "Starting up");

    struct sigaction sa = {0};
    sa.sa_handler = handle_sigterm;
    sigaction(SIGTERM, &sa, NULL);
    sa.sa_handler = handle_sighup;
    sigaction(SIGHUP, &sa, NULL);

    while (g_running) {
        // Do work
        syslog(LOG_DEBUG, "Doing work...");
        sleep(1);
    }

    syslog(LOG_INFO, "Shutting down");
    closelog();
    return 0;
}
```

With `Type=simple` in the unit file, systemd monitors this process directly — no daemonization needed.

### Static Binaries with musl libc

glibc (the standard C library) is not ideal for static linking — it's large and has version compatibility issues. musl libc is a clean, small C library designed for static linking:

```bash
# Install musl compiler wrapper
sudo apt install musl-tools

# Build a static binary
musl-gcc -static -Wall -O2 program.c -o program_static

# Verify: no dynamic dependencies
ldd program_static
# output: not a dynamic executable

# Check size
ls -lh program_static
```

A static musl binary runs on any Linux system with the same architecture, regardless of what libc version is installed. This makes deployment trivial.

Use case: deploying to containers, embedded systems, or systems where you cannot control the installed libraries.

### Docker for C Applications

Docker packages your application with its dependencies into an image:

**Multi-stage Dockerfile (build + minimal runtime):**

```dockerfile
# Stage 1: Build
FROM gcc:13 AS builder

WORKDIR /build
COPY . .
RUN gcc -Wall -O2 -static -o server main.c

# Stage 2: Minimal runtime image
FROM scratch

COPY --from=builder /build/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

The `scratch` base image is empty — your static binary is the entire container. Resulting image is just the size of your binary.

```bash
docker build -t myserver .
docker run -p 8080:8080 myserver
```

For programs that need libc (dynamic):

```dockerfile
FROM debian:bookworm-slim AS builder
RUN apt-get update && apt-get install -y gcc
WORKDIR /build
COPY . .
RUN gcc -Wall -O2 -o server main.c

FROM debian:bookworm-slim
COPY --from=builder /build/server /usr/bin/server
EXPOSE 8080
CMD ["/usr/bin/server"]
```

### Cross-Compilation

Build for a different architecture than your host:

```bash
# Install cross-compiler
sudo apt install gcc-aarch64-linux-gnu

# Build for ARM64
aarch64-linux-gnu-gcc -static program.c -o program_arm64

# Verify target architecture
file program_arm64
# program_arm64: ELF 64-bit LSB executable, ARM aarch64
```

Use cases: building firmware for embedded Linux boards (Raspberry Pi, etc.), building for servers with different architectures, CI systems that build release artifacts for multiple targets.

With CMake:
```bash
cmake -DCMAKE_C_COMPILER=aarch64-linux-gnu-gcc \
      -DCMAKE_SYSTEM_NAME=Linux \
      -DCMAKE_SYSTEM_PROCESSOR=aarch64 \
      ..
```

### Release Checklist

Before distributing a Linux C application:

- [ ] Compile with `-Wall -Wextra -Werror` — zero warnings
- [ ] Run under AddressSanitizer and fix all errors
- [ ] Run under valgrind — zero leaks
- [ ] Strip debug symbols for release: `strip --strip-debug binary`
- [ ] Check all error paths — every system call can fail
- [ ] Handle `SIGTERM` cleanly — flush buffers, close connections
- [ ] Log to syslog or stderr (not stdout) for daemon processes
- [ ] Run as a non-root user
- [ ] Apply systemd security hardening directives
- [ ] Test the package install/remove cycle

## Exercises

### Exercise 1: Package Your Echo Server

Take the TCP echo server from Unit 15 and package it as a `.deb`:
1. Build the binary with `-O2`
2. Strip it: `strip --strip-debug server`
3. Create the DEBIAN control file with correct metadata
4. Build and install the `.deb`
5. Verify it installed: `dpkg -L myserver`

### Exercise 2: systemd Service

Write a systemd unit file for your echo server:
1. Start the service on boot
2. Restart it automatically if it crashes (test by sending `kill -9`)
3. Run it as a dedicated non-root user (`adduser --system echoserver`)
4. View its logs with `journalctl -u echoserver -f`
5. Implement `SIGTERM` handling in the server so it shuts down cleanly

### Exercise 3: Static Binary with musl

Build the echo server as a static binary using musl-gcc:
1. Verify `ldd` shows no dependencies
2. Copy the binary to a Docker `scratch` container and verify it runs
3. Compare the binary size: glibc dynamic vs musl static
4. Use `strip` and measure the size reduction

### Exercise 4: Multi-Stage Docker Build

Write a Dockerfile that:
1. Uses `gcc:13` as the builder image
2. Compiles the echo server statically with musl (install `musl-tools`)
3. Copies the binary into a `scratch` image
4. Exposes the correct port
5. Has a health check: `HEALTHCHECK CMD nc -z localhost 9000`

Build and run it. Connect with `nc localhost 9000`.

### Exercise 5: Cross-Compile for ARM64

Cross-compile the echo server for ARM64:
1. Install `gcc-aarch64-linux-gnu`
2. Build a static binary targeting aarch64
3. Verify the architecture with `file` and `readelf -h`
4. Run it in a Docker container: `docker run --platform linux/arm64 -v $(pwd):/app arm64v8/ubuntu /app/server`

(If you don't have an ARM64 machine, QEMU or Docker Desktop can run the container.)

---

## Curriculum Complete

Congratulations — you have worked through the full Linux native developer curriculum.

You can now:
- Understand how memory works at the hardware level (Unit 01)
- Navigate and automate the Linux environment (Unit 02)
- Write C programs with full understanding of the language (Unit 05)
- Build multi-file projects with Makefiles and CMake (Unit 06)
- Manage memory correctly with malloc/free and detect bugs with valgrind (Unit 07)
- Read and write files using the POSIX file API (Unit 09)
- Create and manage processes, handle signals (Unit 11)
- Write multithreaded programs with proper synchronization (Unit 13)
- Debug and profile programs with gdb, valgrind, strace, and perf (Unit 14)
- Build networked applications with TCP sockets and epoll (Unit 15)
- Create and consume shared and static libraries (Unit 16)
- Package and deploy programs as system services (Unit 18)

### Where to Go Next

- **Linux kernel internals**: Robert Love's *Linux Kernel Development*
- **Advanced C**: *Expert C Programming* by Peter van der Linden
- **Performance engineering**: Brendan Gregg's *Systems Performance*
- **Network programming depth**: *Unix Network Programming* by Stevens
- **Security**: learn about seccomp, namespaces, and capabilities for hardening
- **Rust on Linux**: a memory-safe systems language that interops with C
