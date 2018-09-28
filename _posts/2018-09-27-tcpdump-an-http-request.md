---
layout: post
title: tcpdump an HTTP request
---

I'm playing a bit with `tcpdump`.

I captured packets from `curl example.com` using the command `tcpdump host example.com`.

<small>(`host example.com` will filter for packets arriving at or departing from `example.com`).</small>


Here's the full output; I'll walk through it in the following sections:
```
01:52:05.998903 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [S], seq 2692572934, win 29200, options [mss 1460,sackOK,TS val 2855236 ecr 0,nop,wscale 7], length 0
01:52:06.023174 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [S.], seq 1460480001, ack 2692572935, win 65535, options [mss 1460], length 0
01:52:06.023215 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1, win 29200, length 0
01:52:06.023380 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [P.], seq 1:76, ack 1, win 29200, length 75: HTTP: GET / HTTP/1.1
01:52:06.023588 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [.], ack 76, win 65535, length 0
01:52:06.058386 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [P.], seq 1:1441, ack 76, win 65535, length 1440: HTTP: HTTP/1.1 200 OK
01:52:06.058404 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1441, win 31240, length 0
01:52:06.068479 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [P.], seq 1441:1593, ack 76, win 65535, length 152: HTTP
01:52:06.068497 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1593, win 34080, length 0
01:52:06.068713 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [F.], seq 76, ack 1593, win 34080, length 0
01:52:06.068853 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [.], ack 77, win 65535, length 0
01:52:06.142297 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [F.], seq 1593, ack 77, win 65535, length 0
01:52:06.142320 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1594, win 34080, length 0
```

## IPs and Ports
In the first packet you can see that my local IP is `10.0.2.15` and `example.com` IP is `93.184.216.34`.

We can see I was assigned port `52546` locally and, as expected, `example.com` is accepting connections on the `http` port.

If we use `-n` option to prevent `tcpdump` from converting "addresses" to names it will show the `http` port numerically as `80`.

## TCP 3-way handshake
```
# curl sends SYN
01:52:05.998903 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [S], seq 2692572934, win 29200, options [mss 1460,sackOK,TS val 2855236 ecr 0,nop,wscale 7], length 0

# example.com responds with SYN-ACK
01:52:06.023174 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [S.], seq 1460480001, ack 2692572935, win 65535, options [mss 1460], length 0

# curl ACKs
01:52:06.023215 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1, win 29200, length 0
```

## HTTP
We can see a basic example of HTTP in action.

### GET / HTTP/1.1
After the connection is established, `curl` sends the HTTP GET request:
```
01:52:06.023380 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [P.], seq 1:76, ack 1, win 29200, length 75: HTTP: GET / HTTP/1.1
```
<small>(use the `-A` option if you want to see the packet in ASCII and `-s <bytes>` to avoid truncation)</small>


`example.com` ACKs the segment containing the request:
```
01:52:06.023588 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [.], ack 76, win 65535, length 0
```

### HTTP/1.1 200 OK
`example.com` proceeds to send its HTTP response; the sent segments interleaved with ACKs (in this particular example):
```
01:52:06.058386 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [P.], seq 1:1441, ack 76, win 65535, length 1440: HTTP: HTTP/1.1 200 OK

01:52:06.058404 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1441, win 31240, length 0

01:52:06.068479 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [P.], seq 1441:1593, ack 76, win 65535, length 152: HTTP

01:52:06.068497 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1593, win 34080, length 0
```

### Aside: Direct Server Response load balancing 
Here's a neat observation about the size of HTTP requests relative to responses.

In this (basic) example, the request was a small 75 bytes and happened to fit into a single TCP segment. 
While the response wasn't particularly large at 1592 bytes, it's still sizeable in comparison to the request.

This pattern holds true for most HTTP communications and makes a [Direct Server Response](https://www.haproxy.com/blog/layer-4-load-balancing-direct-server-return-mode/) a desirable feature if using a load balancer; only the (small) requests will traverse the load balancer.

## Round trip time and throughput
We can get a rough estimate of RTT by comparing the timestamps of when a segment was sent and when it was acknowledged.

We can similarly get a rough estimate of throughput by examining the total data sent over the life of the connection.

These numbers would be more useful if there was a larger sample.


## TCP connection termination

```
# curl initiates connection termination with a FIN
01:52:06.068713 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [F.], seq 76, ack 1593, win 34080, length 0

# example.com ACKs and then sends its FIN
01:52:06.068853 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [.], ack 77, win 65535, length 0
01:52:06.142297 IP 93.184.216.34.http > 10.0.2.15.52546: Flags [F.], seq 1593, ack 77, win 65535, length 0

# curl ACKs
01:52:06.142320 IP 10.0.2.15.52546 > 93.184.216.34.http: Flags [.], ack 1594, win 34080, length 0
```

