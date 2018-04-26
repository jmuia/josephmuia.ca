---
layout: post 
title: Connected UDP Sockets and ICMP Errors
---

I wrote a basic implementation of [`traceroute`](https://github.com/jmuia/traceroute). I decided to keep it minimal and low-effort. I understand how traceroute works and just wanted a small project to refresh my C network programming.

I learned about "connected" UDP sockets in [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) and thought I could cheat a little bit. Rather than fussing with a raw ICMP socket I could just use a connected UDP socket and let the kernel figure out what ICMP error messages belong to it.

The rest of this post is basically the dialogue I had with myself while trying to understand why it wasn't working as expected. 


## ICMP errors are propagated to connected UDP sockets

I found [various](http://www.softlab.ntua.gr/facilities/documentation/unix/unix-socket-faq/unix-socket-faq-5.html) [resources](https://www.ietf.org/mail-archive/web/behave/current/msg10927.html) indicating that ICMP error messages are propagated on connected UDP sockets.

I wrote a small bit of code (below) to test out getting the ICMP error message for an unreachable destination port and used netcat to run a process listening for UDP packets `nc -lu 9000`.

I created a connected UDP socket to `send()` a packet to a different port and then called `recv()`. 

```
tty000 $ ./send_udp_recv_icmp localhost 33434
recv error: Connection refused
```

It works! I could also see the ICMP error message in Wireshark.

I confirmed that there were no errors when I sent packets to port 9000 and tested against a remote host as well.
```
tty999 $ nc -lu 9000
```
```
tty000 $ ./send_udp_recv_icmp localhost 9000
```
```
tty999 receives "Hello, World!" and hits enter.
```
```
tty000 $ <exited successfully after response>
```


## Or are they?

Next up: ICMP time-exceeded messages

The code that follows is a pared down example with no error-checking, but should demonstrate what I was attempting.

```
/* send_udp_recv_icmp.c */

#include ...

#define MAXDATASIZE 1500

int main(int argc, char *argv[]) {
  int sockfd;
  int ttl;
  char buf[MAXDATASIZE];
  char message[] = "Hello, world!";
  struct addrinfo *ai;

  if (argc != 3) {
    fprintf(stderr, "usage: hostname port\n");
    exit(1);
  }

  getaddrinfo(argv[1], argv[2], NULL, &ai);
  sockfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
  connect(sockfd, ai->ai_addr, ai->ai_addrlen);

  ttl = 1;
  setsockopt(sockfd, IPPROTO_IP, IP_TTL, &ttl, sizeof(ttl));
  if ((send(sockfd, message, sizeof(message), 0)) == -1) {
    perror("send error");
    exit(1);
  }

  // I expect this to fail if send() resulted in an ICMP
  // error message but instead it hangs.
  if ((recv(sockfd, buf, MAXDATASIZE - 1, 0)) == -1) {
    perror("recv error");
    exit(1);
  }

  return 0;
}
```

It just seemed to hang at `recv()` (i.e. it didn't receive a response). Shouldn't the error propogate? I tried sending more packets as well. Nothing.

I checked Wireshark for my outgoing UDP packet and for incoming ICMP time-exceeded messages.
```
19	3.121938	192.168.2.18	172.217.1.14	UDP	56	57501 → 33434 Len=14
```
```
20	3.124019	192.168.2.1	192.168.2.18	ICMP	84	Time-to-live exceeded (Time to live exceeded in transit)
```

There they are. I can tell the ICMP error message is for my socket by matching the source port of the UDP packet, `57501`, to the source port found in the UDP header inside the ICMP message\*. Same for the destination port `33434`. <small>(\*assuming the source port is not being used by another socket for the same purposes).</small>

```
Frame 20: 84 bytes on wire (672 bits), 84 bytes captured (672 bits) on interface 0
Ethernet II, Src: ...
Internet Protocol Version 4, Src: 192.168.2.1, Dst: 192.168.2.18
Internet Control Message Protocol
    Type: 11 (Time-to-live exceeded)
    Code: 0 (Time to live exceeded in transit)
    Checksum: 0x65c9 [correct]
    [Checksum Status: Good]
    Internet Protocol Version 4, Src: 192.168.2.18, Dst: 172.217.1.14
    User Datagram Protocol, Src Port: 57501, Dst Port: 33434
        Source Port: 57501 // this matches!
        Destination Port: 33434 // this matches!
        Length: 22
        Checksum: 0xea9b [unverified]
        [Checksum Status: Unverified]
        [Stream index: 1]
    Data (14 bytes)
```

With TTL = 1 it looks like my router is sending the ICMP error messages.


## Socket options

While poking around various manpages and Google/StackOverflow, I came across `SO_ERROR` in the [FreeBSD manpages](https://www.freebsd.org/cgi/man.cgi?query=getsockopt):

> SO_ERROR returns any pending error on the socket and clears the error
> status. It may be used to check for asynchronous errors on connected
> datagram sockets or for other asynchronous errors. 

Sounds exactly like what I wanted! But, unfortunately, this didn't help. I tried adding a `sleep()` before calling `getsockopt()` with `SO_ERROR` in case it was called too fast, but still no luck.

Linux has an option `IP_RECVERR` but I wanted my implementation to be portable.


## Checking the manpages and RFCs

The Linux manpages for [UDP](http://man7.org/linux/man-pages/man7/udp.7.html) indicate:
> All fatal errors will be passed to the user as an error return even
> when the socket is not connected.

Maybe it's not considered "fatal"?

That manpage has a reference to [RFC 1122](https://tools.ietf.org/html/rfc1122) _Requirements for Internet Hosts -- Communication Layers_.

on page 40:
> An incoming Time Exceeded message MUST be passed to the
> transport layer.

on page 77:
> UDP MUST pass to the application layer all ICMP error
> messages that it receives from the IP layer. 

[BCP 145](https://tools.ietf.org/html/bcp145) _UDP Usage Guidelines_ also mentions on page 32:
> On some stacks, a bound socket
> also allows an application to be notified when ICMP error messages
> are received for its transmissions [RFC1122].


## What am I missing?

Well, I decided to stop spending time on this particular issue and ended up using a raw ICMP socket to receive error messages.

There is a logical explanation to be heard. If you know what it is, please get in touch with me. I'd be happy to hear it.

