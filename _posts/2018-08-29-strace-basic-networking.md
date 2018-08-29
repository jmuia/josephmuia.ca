---
layout: post
title: strace basic networking
---

More `strace`... this time, we're going to spy on `netcat`.

If you've done a bit of basic (TCP) network programming, you'll recognize the typical pattern for clients as:
1. `socket()`: get a socket
2. `connect()`: connect to the server
3. `send()` / `recv()`: send and receive things!

and for servers as:
1. `socket()`: get a socket
2. `bind()`: bind a port
3. `listen()`: listen for incoming connections and add them to queue
4. `accept()`: block until an incoming connection is accepted
5. `send()` / `recv()`: send and receive things!

## Client

Let's see it in action for the client:
```
jmuia@jmuia:~$ strace -f nc 0 4000
```
```
execve("/bin/nc", ["nc", "0", "4000"], [/* 57 vars */]) = 0
[ ... ]
open("/etc/services", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=19605, ...}) = 0
read(3, "# Network services, Internet sty"..., 4096) = 4096
read(3, "\t\t# IPX\nipx\t\t213/udp\nimap3\t\t220/"..., 4096) = 4096
read(3, "nessus\t\t1241/tcp\t\t\t# Nessus vuln"..., 4096) = 4096
read(3, "347/tcp\t\t\t# gnutella\ngnutella-rt"..., 4096) = 4096
read(3, "ureg\t779/udp\t\tmoira_ureg\t# Moira"..., 4096) = 3221
read(3, "", 4096)                       = 0
close(3)                                = 0
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
fcntl(3, F_GETFL)                       = 0x2 (flags O_RDWR)
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_INET, sin_port=htons(4000), sin_addr=inet_addr("0.0.0.0")}, 16) = -1 EINPROGRESS (Operation now in progress)
select(4, NULL, [3], NULL, NULL)        = 1 (out [3])
getsockopt(3, SOL_SOCKET, SO_ERROR, [111], [4]) = 0
fcntl(3, F_SETFL, O_RDWR)               = 0
close(3)                                = 0
close(-1)                               = -1 EBADF (Bad file descriptor)
exit_group(1)                           = ?
+++ exited with 1 +++
```

We can see that `netcat` opens and reads the `/etc/services` file, containing port numbers of well known services.
```
open("/etc/services", O_RDONLY|O_CLOEXEC) = 3
...
read(3, "# Network services, Internet sty"..., 4096) = 4096
...
```

we can see the call to get a TCP socket:
```
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
```

and then connecting to `0.0.0.0:4000`:
```
connect(3, {sa_family=AF_INET, sin_port=htons(4000), sin_addr=inet_addr("0.0.0.0")}, 16) = -1 EINPROGRESS (Operation now in progress)
```

we can also see that `netcat` uses a non-blocking socket (`O_NONBLOCK`) and `select()`:
```
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
...
select(4, NULL, [3], NULL, NULL)        = 1 (out [3])
getsockopt(3, SOL_SOCKET, SO_ERROR, [111], [4]) = 0
...
```

## Server

```
jmuia@jmuia:~$ strace -f nc -l 0 4000
```
```
execve("/bin/nc", ["nc", "-l", "0", "4000"], [/* 57 vars */]) = 0
[ ... ]
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(3, {sa_family=AF_INET, sin_port=htons(4000), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 1)                            = 0
accept(3, {sa_family=AF_INET, sin_port=htons(34836), sin_addr=inet_addr("127.0.0.1")}, [16]) = 4
close(3)                                = 0
poll([{fd=4, events=POLLIN}, {fd=0, events=POLLIN}], 2, -1) = 1 ([{fd=4, revents=POLLIN}])
read(4, "hi! bye!\n", 2048)             = 9
write(1, "hi! bye!\n", 9hi! bye!
)               = 9
poll([{fd=4, events=POLLIN}, {fd=0, events=POLLIN}], 2, -1) = 1 ([{fd=4, revents=POLLIN}])
read(4, "", 2048)                       = 0
shutdown(4, SHUT_RD)                    = 0
close(4)                                = 0
close(3)                                = -1 EBADF (Bad file descriptor)
close(3)                                = -1 EBADF (Bad file descriptor)
exit_group(0)                           = ?
+++ exited with 0 +++
```

We can see the server's call to get a TCP socket:
```
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
```

and setting the option to reuse the port:
```
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
```

then binding port `4000` and listening for connections:
```
bind(3, {sa_family=AF_INET, sin_port=htons(4000), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 1)                            = 0
```

`accept()` blocks and looks like `accept (3, ` until I connect from another terminal (note the new file descriptor 4!):
```
accept(3, {sa_family=AF_INET, sin_port=htons(34836), sin_addr=inet_addr("127.0.0.1")}, [16]) = 4
```

it then polls until receiving the message I sent from another terminal ("hi! bye!\n"), reads the data, and writes it back to stdout:
```
poll([{fd=4, events=POLLIN}, {fd=0, events=POLLIN}], 2, -1) = 1 ([{fd=4, revents=POLLIN}])
read(4, "hi! bye!\n", 2048)             = 9
write(1, "hi! bye!\n", 9hi! bye!
)               = 9
```

finally, shutting down and closing as the client disconnects.

