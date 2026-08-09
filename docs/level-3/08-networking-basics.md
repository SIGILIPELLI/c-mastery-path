# 08 · Networking Basics

A socket is a file descriptor that happens to have another machine on the
other end. Once you have one, `read`/`write` (or their socket-flavoured
cousins `recv`/`send`) work much like they do on a file — which is the good
news. The bad news is everything around it: addresses have to be in the
right byte order, connections take four calls to set up, and TCP will
happily hand you *half* a message because it has no idea your bytes were
ever grouped into messages at all.

This module builds a working echo server, talks to it from a C client, then
serves real HTTP to `curl`.

## The four calls that make a server

A TCP server always follows the same sequence:

| Call | What it does |
|---|---|
| `socket()` | Create an unbound endpoint, get an fd back |
| `bind()` | Claim a local address and port |
| `listen()` | Start queueing incoming connections |
| `accept()` | Pull one connection off the queue as a *new* fd |

`accept` is the one people misread. It does not return data — it returns a
**second file descriptor** representing one client. The listening socket
stays open, ready for the next `accept`.

```c
// echo_server.c -- accept one client at a time and echo back what it sends
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 9090

int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    if (srv < 0) { perror("socket"); return 1; }

    int yes = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);

    struct sockaddr_in addr = {0};
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);   // every local interface
    addr.sin_port        = htons(PORT);         // network byte order!

    if (bind(srv, (struct sockaddr *)&addr, sizeof addr) < 0) {
        perror("bind"); return 1;
    }
    if (listen(srv, 16) < 0) { perror("listen"); return 1; }
    printf("listening on port %d\n", PORT);
    fflush(stdout);

    for (;;) {
        struct sockaddr_in peer;
        socklen_t plen = sizeof peer;
        int c = accept(srv, (struct sockaddr *)&peer, &plen);
        if (c < 0) { perror("accept"); continue; }

        char ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &peer.sin_addr, ip, sizeof ip);
        printf("connection from %s:%u\n", ip, ntohs(peer.sin_port));
        fflush(stdout);

        char buf[256];
        ssize_t n;
        while ((n = recv(c, buf, sizeof buf, 0)) > 0) {
            printf("received %zd bytes\n", n);
            fflush(stdout);
            ssize_t sent = 0;
            while (sent < n) {                  // send may be partial too
                ssize_t w = send(c, buf + sent, (size_t)(n - sent), 0);
                if (w <= 0) { perror("send"); break; }
                sent += w;
            }
        }
        close(c);
        printf("client disconnected\n");
        fflush(stdout);
    }
}
```

`SO_REUSEADDR` deserves a note. After a connection closes, the kernel keeps
the address in `TIME_WAIT` for a couple of minutes; without this option a
restarted server gets `bind: Address already in use` and refuses to start.
Every server you write should set it.

## The client side

Clients skip `bind` and `listen` entirely — `connect` picks a local port
automatically:

```c
// echo_client.c -- connect, send a line, read the echo back
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main(int argc, char **argv) {
    const char *msg = (argc > 1) ? argv[1] : "hello sockets";

    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) { perror("socket"); return 1; }

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(9090);
    if (inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr) != 1) {
        fprintf(stderr, "bad address\n"); return 1;
    }

    if (connect(fd, (struct sockaddr *)&addr, sizeof addr) < 0) {
        perror("connect"); return 1;
    }

    size_t len = strlen(msg);
    if (send(fd, msg, len, 0) < 0) { perror("send"); return 1; }

    char buf[256];
    ssize_t n = recv(fd, buf, sizeof buf - 1, 0);
    if (n < 0) { perror("recv"); return 1; }
    buf[n] = '\0';                              // recv does NOT null-terminate
    printf("sent:     \"%s\" (%zu bytes)\n", msg, len);
    printf("echoed:   \"%s\" (%zd bytes)\n", buf, n);

    close(fd);
    return 0;
}
```

Build both and run the server in the background:

```bash
clang -Wall -Wextra -o echo_server echo_server.c
clang -Wall -Wextra -o echo_client echo_client.c
./echo_server &
./echo_client "hello sockets"
./echo_client "second connection"
```

```text
sent:     "hello sockets" (13 bytes)
echoed:   "hello sockets" (13 bytes)
sent:     "second connection" (17 bytes)
echoed:   "second connection" (17 bytes)
```

And on the server's side of the conversation:

```text
listening on port 9090
connection from 127.0.0.1:53433
received 13 bytes
client disconnected
connection from 127.0.0.1:53434
received 17 bytes
client disconnected
```

Two clients, two different ephemeral source ports (53433, 53434) chosen by
the kernel. The `recv` loop ended each time because a closed connection
makes `recv` return **0** — that is end-of-stream, not an error. A return of
`-1` is the error case, and the two must never be conflated.

## Byte order is not optional

Network protocols fixed on big-endian ("network byte order") decades ago.
Most machines you will run on today are little-endian, so the conversion is
a real byte swap, not a no-op:

```c
// byteorder.c -- what htons/htonl actually do on this machine
#include <stdio.h>
#include <arpa/inet.h>

static void dump(const char *label, const void *p, size_t n) {
    const unsigned char *b = p;
    printf("%-18s", label);
    for (size_t i = 0; i < n; i++) printf(" %02x", b[i]);
    putchar('\n');
}

int main(void) {
    uint16_t port = 9090;
    uint16_t net_port = htons(port);
    uint32_t ip = 0x7f000001;             // 127.0.0.1
    uint32_t net_ip = htonl(ip);

    printf("port host order   = %u\n", port);
    dump("  bytes in memory:", &port, sizeof port);
    printf("port network order= %u\n", net_port);
    dump("  bytes in memory:", &net_port, sizeof net_port);

    dump("ip host order:", &ip, sizeof ip);
    dump("ip network order:", &net_ip, sizeof net_ip);

    printf("ntohs(htons(9090)) = %u\n", ntohs(htons(port)));
    return 0;
}
```

```text
port host order   = 9090
  bytes in memory: 82 23
port network order= 33315
  bytes in memory: 23 82
ip host order:     01 00 00 7f
ip network order:  7f 00 00 01
ntohs(htons(9090)) = 9090
```

Port 9090 stored as `82 23` in memory becomes `23 82` on the wire. Forget
`htons` and you bind to port 33315 instead — the program starts happily and
your client just can't find it. Note also that `htons` returning `33315` is
*not* a bug: that is what 9090's swapped bytes read as when reinterpreted as
a host-order integer. Never print or compare a network-order value as a
number; only pass it to the socket API.

## TCP is a byte stream, not a message queue

This is the single biggest source of networking bugs in C. TCP guarantees
that your bytes arrive in order and without corruption. It guarantees
*nothing* about how they are grouped.

```c
// stream_demo.c -- TCP has no message boundaries: three sends, one recv
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main(void) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in a = {0};
    a.sin_family = AF_INET;
    a.sin_port   = htons(9090);
    inet_pton(AF_INET, "127.0.0.1", &a.sin_addr);
    if (connect(fd, (struct sockaddr *)&a, sizeof a) < 0) {
        perror("connect"); return 1;
    }

    const char *parts[] = { "ALPHA", "BETA", "GAMMA" };
    for (int i = 0; i < 3; i++) {
        send(fd, parts[i], strlen(parts[i]), 0);
        printf("send #%d: \"%s\"\n", i + 1, parts[i]);
    }

    char buf[256];
    ssize_t n = recv(fd, buf, sizeof buf - 1, 0);
    buf[n] = '\0';
    printf("one recv returned %zd bytes: \"%s\"\n", n, buf);
    close(fd);
    return 0;
}
```

```text
send #1: "ALPHA"
send #2: "BETA"
send #3: "GAMMA"
one recv returned 14 bytes: "ALPHABETAGAMMA"
```

Meanwhile the echo server logged:

```text
connection from 127.0.0.1:53444
received 5 bytes
received 9 bytes
```

Three `send` calls arrived as **two** reads on the server (5 bytes, then
`BETA` and `GAMMA` merged into 9) and came back to the client as **one**
14-byte blob. Nothing here is broken — this is TCP working exactly as
specified.

The consequence: you must define your own framing. The usual choices are a
delimiter (HTTP uses `\r\n\r\n` to end headers) or a length prefix (send a
4-byte `htonl` length, then that many bytes). Whichever you pick, `recv`
must run in a loop that accumulates until a complete frame is present.

## Serving real HTTP

HTTP over TCP is just text with a required shape: a request line, headers, a
blank line, then the body. That is little enough that `curl` will talk to
about eighty lines of C:

```c
// tiny_http.c -- speak just enough HTTP/1.1 for curl to be happy
static void serve(int c) {
    char req[2048];
    ssize_t n = recv(c, req, sizeof req - 1, 0);
    if (n <= 0) return;
    req[n] = '\0';

    char method[16] = "", path[256] = "";
    sscanf(req, "%15s %255s", method, path);    // width limits are the point
    printf("%s %s\n", method, path);
    fflush(stdout);

    char body[512];
    int blen = snprintf(body, sizeof body,
                        "{\"method\":\"%s\",\"path\":\"%s\"}\n", method, path);

    char head[512];
    int hlen = snprintf(head, sizeof head,
                        "HTTP/1.1 200 OK\r\n"
                        "Content-Type: application/json\r\n"
                        "Content-Length: %d\r\n"
                        "Connection: close\r\n"
                        "\r\n", blen);
    send(c, head, (size_t)hlen, 0);
    send(c, body, (size_t)blen, 0);
}
```

`main` is the same `socket`/`bind`/`listen`/`accept` loop as the echo
server, on port 8080, calling `serve(c)` then `close(c)`.

```bash
clang -Wall -Wextra -o tiny_http tiny_http.c
./tiny_http &
curl -s http://localhost:8080/hello
curl -s -i http://localhost:8080/status
```

```text
{"method":"GET","path":"/hello"}
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 34
Connection: close

{"method":"GET","path":"/status"}
```

Those `%15s` and `%255s` width limits in the `sscanf` are load-bearing
security, not tidiness. A bare `%s` writing into `method[16]` from an
attacker-controlled request line is a stack buffer overflow reachable by
anyone who can open a socket — [Level 4's security
module](../level-4/05-security-in-c.md) takes that apart properly. Note too
that this server reads the request with a **single** `recv`, which the
previous section just showed is unsafe: a client that dribbles its request
line out in pieces would be mis-parsed. Real servers loop until they see the
blank line that ends the headers.

## Cheat sheet

| Call | Purpose |
|---|---|
| `socket(AF_INET, SOCK_STREAM, 0)` | New TCP/IPv4 endpoint (`SOCK_DGRAM` for UDP) |
| `bind(fd, addr, len)` | Claim a local port; needs `SO_REUSEADDR` to restart cleanly |
| `listen(fd, backlog)` | Begin queueing connections |
| `accept(fd, &peer, &len)` | Return a **new** fd for one client |
| `connect(fd, addr, len)` | Client-side: reach out to a server |
| `recv(fd, buf, n, 0)` | Read; `0` means peer closed, `-1` means error |
| `send(fd, buf, n, 0)` | Write; may send **fewer** bytes than asked — loop |
| `close(fd)` | Ends the connection (sends FIN) |
| `htons` / `htonl` | Host → network byte order (16-bit / 32-bit) |
| `ntohs` / `ntohl` | Network → host byte order |
| `inet_pton` / `inet_ntop` | Text address ↔ binary; the `_ntoa`/`_addr` versions are legacy |
| `setsockopt(..., SO_REUSEADDR, ...)` | Rebind a port still in `TIME_WAIT` |

Traps worth re-reading before you debug for an hour:

- `recv` does **not** null-terminate. Always `buf[n] = '\0'` with room
  reserved (`sizeof buf - 1`).
- `send` returning less than `n` is normal on a busy socket. The loop in
  `echo_server.c` is the minimum correct form.
- Writing to a socket whose peer has closed raises `SIGPIPE`, killing your
  process by default. Servers set `signal(SIGPIPE, SIG_IGN)` and check
  `errno == EPIPE` instead.
- `struct sockaddr_in` must be zeroed before use; `= {0}` does it. Leftover
  stack garbage in the padding has caused real bind failures.

## Exercise

Give the echo server **length-prefixed framing** so message boundaries
survive the stream. The client sends a 4-byte `htonl` length followed by
exactly that many bytes; the server reads the 4-byte header in a loop, then
reads exactly that many payload bytes in a second loop, and echoes back the
same framing.

Prove it works by sending three messages back to back from a single
`connect` — the same pattern that produced `ALPHABETAGAMMA` above — and
confirming the server now reports three separate messages of the right
lengths. Then deliberately break it: have the client send the 4-byte length
and then only *half* the payload before sleeping two seconds. A correct
implementation blocks in its read loop and completes the message when the
rest arrives; a naive one returns a short, corrupt message immediately.
Finally, send a length prefix of `0xFFFFFFFF` and make sure your server
rejects it instead of trying to `malloc` four gigabytes.
