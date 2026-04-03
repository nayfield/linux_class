# Unit 15: Networking & Sockets

## Overview

Network programming on Linux uses the same file descriptor model as files and pipes. A socket is just a file descriptor you can `read` and `write`. The BSD socket API, designed in the 1980s, remains the foundation of all networked software on Linux. This unit covers TCP sockets from creating a connection to building a server that handles thousands of concurrent clients using `epoll`. After this unit you will understand how every networked program on your system works at the system call level.

## Prerequisites

- Unit 09 (Filesystem & File I/O) — file descriptors, read/write, blocking I/O
- Unit 11 (Processes & Signals) — fork, non-blocking patterns
- Unit 13 (Threads & Synchronization) — multi-threaded servers

## Learning Objectives

- Understand the TCP/IP model at a practical level
- Create TCP client and server sockets
- Perform address resolution with `getaddrinfo`
- Send and receive data reliably over a stream socket
- Set socket options (reuse address, keep-alive, timeouts)
- Implement a concurrent server with `fork`, threads, or `epoll`
- Use `select`, `poll`, and `epoll` for I/O multiplexing
- Understand non-blocking sockets and how to use them

## Reading / Resources

- `man 2 socket`, `man 2 bind`, `man 2 listen`, `man 2 accept`
- `man 2 connect`, `man 2 send`, `man 2 recv`
- `man 3 getaddrinfo`
- `man 7 ip`, `man 7 tcp`, `man 7 socket`
- `man 7 epoll`
- Beej's Guide to Network Programming: https://beej.us/guide/bgnet/

## Concepts

### The TCP/IP Model

| Layer | Protocols | What it does |
|-------|-----------|-------------|
| Application | HTTP, SSH, DNS | Your application protocol |
| Transport | TCP, UDP | End-to-end delivery |
| Network | IP | Routing between hosts |
| Link | Ethernet, WiFi | Physical transmission |

**TCP** (Transmission Control Protocol) provides:
- Reliable, ordered delivery (no data loss, no reordering)
- Connection-oriented (explicit connect/accept)
- Stream-based (no message boundaries)

**UDP** provides unreliable datagram delivery — simpler and lower latency.

### Socket Basics

A socket is an endpoint for communication. Creating a socket:

```c
#include <sys/socket.h>

int sock = socket(AF_INET,    // address family: IPv4
                  SOCK_STREAM, // type: TCP stream
                  0);          // protocol: auto-select

if (sock == -1) { perror("socket"); exit(1); }
```

For IPv6: `AF_INET6`. For UDP: `SOCK_DGRAM`.

### TCP Server: Bind, Listen, Accept

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>

int server_fd = socket(AF_INET, SOCK_STREAM, 0);

// Allow immediate reuse of the port after restart
int opt = 1;
setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

// Bind to port 8080 on all interfaces
struct sockaddr_in addr = {0};
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = INADDR_ANY;
addr.sin_port = htons(8080);  // htons: host-to-network byte order

if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
    perror("bind"); exit(1);
}

// Start listening (queue up to 10 pending connections)
listen(server_fd, 10);

// Accept connections in a loop
while (1) {
    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);
    int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &len);

    char ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &client_addr.sin_addr, ip, sizeof(ip));
    printf("Connection from %s:%d\n", ip, ntohs(client_addr.sin_port));

    // Handle client_fd...
    close(client_fd);
}
```

### TCP Client: Connect

```c
#include <sys/socket.h>
#include <netdb.h>

struct addrinfo hints = {0}, *res;
hints.ai_family = AF_UNSPEC;      // IPv4 or IPv6
hints.ai_socktype = SOCK_STREAM;

// Resolve hostname and port
getaddrinfo("example.com", "80", &hints, &res);

int sock = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
connect(sock, res->ai_addr, res->ai_addrlen);

freeaddrinfo(res);

// Now send/recv on sock
```

Always use `getaddrinfo` — it handles IPv4/IPv6 transparently and returns all available addresses.

### Sending and Receiving

```c
// Send — like write but with flags
ssize_t sent = send(sock, buf, len, 0);

// Recv — like read but with flags
ssize_t received = recv(sock, buf, sizeof(buf), 0);
// Returns 0 when peer closed the connection
```

**TCP is a stream** — there are no message boundaries. A single `send("hello")` may arrive as `"hel"` + `"lo"` on the other side. Always loop until all bytes are sent:

```c
ssize_t send_all(int fd, const void *buf, size_t len) {
    size_t total = 0;
    while (total < len) {
        ssize_t n = send(fd, (char*)buf + total, len - total, 0);
        if (n <= 0) return -1;
        total += n;
    }
    return total;
}
```

### Byte Order

Network protocols use **big-endian** (most significant byte first). x86 CPUs use little-endian. Always convert:

```c
htons(port)   // host to network short (16-bit)
htonl(addr)   // host to network long  (32-bit)
ntohs(port)   // network to host short
ntohl(addr)   // network to host long
```

### Handling Multiple Clients: Fork Model

Simple but doesn't scale to thousands of connections:

```c
while (1) {
    int client_fd = accept(server_fd, NULL, NULL);
    pid_t pid = fork();
    if (pid == 0) {
        close(server_fd);  // child doesn't need the listening socket
        handle_client(client_fd);
        close(client_fd);
        exit(0);
    }
    close(client_fd);  // parent doesn't need client socket
}
```

### I/O Multiplexing: select and poll

Monitor multiple file descriptors for readiness:

```c
#include <sys/select.h>

fd_set readfds;
FD_ZERO(&readfds);
FD_SET(server_fd, &readfds);
FD_SET(client_fd, &readfds);

struct timeval timeout = {.tv_sec = 5};
int ready = select(max_fd + 1, &readfds, NULL, NULL, &timeout);

if (FD_ISSET(server_fd, &readfds)) { /* new connection */ }
if (FD_ISSET(client_fd, &readfds)) { /* client sent data */ }
```

`select` is limited to 1024 FDs and is O(n) — fine for small servers. Use `poll` for cleaner API and slightly higher limits, or `epoll` for high-performance servers.

### epoll — Scalable I/O

`epoll` is Linux-specific but scales to millions of connections. It's O(1) for events regardless of the number of monitored FDs:

```c
#include <sys/epoll.h>

int epfd = epoll_create1(0);

// Register a file descriptor
struct epoll_event ev = {
    .events = EPOLLIN,        // notify when data available to read
    .data.fd = server_fd
};
epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

// Event loop
struct epoll_event events[64];
while (1) {
    int nfds = epoll_wait(epfd, events, 64, -1);  // -1 = block forever
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == server_fd) {
            // Accept new connection and add to epoll
            int client_fd = accept(server_fd, NULL, NULL);
            ev.data.fd = client_fd;
            ev.events = EPOLLIN | EPOLLET;  // edge-triggered
            epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
        } else {
            // Handle client data
            handle_client(events[i].data.fd);
        }
    }
}
```

Edge-triggered (`EPOLLET`) mode: notify only when the FD state changes (new data arrives), not repeatedly while data is available. Requires reading until `EAGAIN`.

### Non-Blocking Sockets

```c
#include <fcntl.h>

int flags = fcntl(sock, F_GETFL, 0);
fcntl(sock, F_SETFL, flags | O_NONBLOCK);

// Now read/write return immediately with EAGAIN/EWOULDBLOCK if no data
ssize_t n = recv(sock, buf, sizeof(buf), 0);
if (n == -1 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
    // No data available right now, try later
}
```

Non-blocking I/O is required for epoll edge-triggered mode and for building responsive event loops.

### Useful Tools for Network Debugging

```bash
nc localhost 8080         # connect to a TCP port (netcat)
nc -l 8080                # listen on a port
curl -v http://localhost:8080/  # test HTTP server
ss -tlnp                  # show listening TCP sockets
ss -tnp                   # show established connections
strace -e trace=network ./server  # trace socket system calls
tcpdump -i lo port 8080   # capture packets on loopback
```

## Exercises

### Exercise 1: TCP Echo Server and Client

**Server:** Listen on port 9000. For each client connection, read data and echo it back verbatim. Handle one client at a time (no concurrency yet).

**Client:** Connect to `localhost:9000`. Read from stdin, send to server, print the response.

Test with `nc`:
```bash
echo "hello world" | nc localhost 9000
```

Verify with strace that the correct system calls are made.

### Exercise 2: Concurrent Echo Server (fork)

Extend Exercise 1 to handle multiple clients simultaneously using `fork`. Each accepted connection spawns a child process. The parent must handle `SIGCHLD` to avoid zombies.

Test by opening multiple terminals and connecting simultaneously.

### Exercise 3: HTTP/1.0 Server

Implement a minimal HTTP/1.0 server that:
- Serves static files from a directory
- Returns `200 OK` with file contents for valid paths
- Returns `404 Not Found` for missing files
- Sets a correct `Content-Length` header
- Handles at least `GET` requests

Test with `curl`:
```bash
curl -v http://localhost:8080/index.html
curl -v http://localhost:8080/nonexistent
```

Parse the request line: `GET /path HTTP/1.0\r\n`

### Exercise 4: epoll Echo Server

Rewrite the echo server from Exercise 1 using a single-threaded `epoll` event loop that handles multiple concurrent clients without forking or threads.

Benchmark: use `ab` (Apache Benchmark) or write your own concurrent test client to measure how many connections per second your server handles:

```bash
# Install: apt install apache2-utils
ab -n 10000 -c 100 http://localhost:9000/
```

### Exercise 5: Simple Chat Server

Build a server where multiple clients can connect and broadcast messages to each other:
- Client sends a line of text
- Server forwards it to all other connected clients, prefixed with a client ID
- Use `epoll` for scalability
- Handle client disconnections cleanly

## What Comes Next

Unit 16 covers libraries and dynamic linking — how to package your code as a shared library (`.so`), how `dlopen` works, and how the dynamic linker loads your program's dependencies at startup.
