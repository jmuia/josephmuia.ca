---
layout: post
title: Network namespaces to the Internet with veth and NAT
---

I've decided to learn a bit about how containers work. My goal here is to connect to the Internet from within a new network namespace.

There were a lot of firsts for me in this post, so if you've got corrections or "well, actually"s, I'm all ears.

I tried to explain most of the commands, but if you want to understand one more thoroughly I recommend checking the man-pages. I found them particularly useful for `ip` and `iptables` commands.

# Namespaces
Namespaces and cgroups are Linux kernel features that serve as the main ingredients for making containers.

Namespaces provide isolated partitions of various kernel resources, which are grouped by namespace type. In this post I'm going to look at the network `net` namespace type, which provides a separate network stack for each namespace.

A quick way to show this is `unshare`. By default, child processes share the same namespaces as their parent. Unshare will run a program and "unshare" some namespaces from the parent.

To run a `<command>` in a new net namespace:
```
unshare --net <command>
``` 

Let's check out the devices in that namespace:
```
root@ubuntu-xenial:/home/vagrant# unshare --net ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

A new network namespace contains only the loopback interface.
<small>A note: it won't be able to connect to the Internet (try pinging example.com `unshare --net ping 93.184.216.34`)</small>

# veth
veth is a term I'd heard at work, but didn't really know what it was.

Turns out it is pretty simple: veth is a virtual ethernet interface.

veth are always created in pairs (kind of like an ethernet cable, with two ends). This is going to be helpful in connecting to the Internet. We can plug in one end of the veth cable to the new network namespace and another to the default namespace.

# Named network namespace
According to `man 8 ip-netns`
> By convention a named network namespace is an object at /var/run/netns/NAME that can be opened.

I'm going to create a new net namespace, `netns0`, for use in this experiment:
```
ip netns add netns0
```

and to check it is there:
```
ip netns list
```

Instead of using unshare, I'm going to use `ip netns exec <namespace> <command>` throughout this post to execute commands in the new namespace.

Our `ping` to example.com is now `ip netns exec netns0 ping 93.184.216.34` and still doesn't work (for now).

### Anonymous network namespaces
A small aside on anonymous network namespaces, which, for example, are created with unshare and are not listed in `ip netns list`.

To demonstrate:
```
# See the namespaces of pid 1
ls -la /proc/1/ns
```
```
# See the namespaces of the current shell process
ls -la /proc/$$/ns
```
```
# See the namespaces of an unshared process
unshare --net bash -c 'ls -la /proc/$$/ns'

# Notice that in the unshared process the net namespace is different
```
```
# It's also not listed in ip netns list
unshare --net bash -c 'ip netns list'
```

# ping localhost
If you try to `ping localhost` you'll see that doesn't work either.
```
root@ubuntu-xenial:/home/vagrant# ip netns exec netns0 ping localhost
connect: Network is unreachable
```

If you review the output of `ip a` inside `netns0`, you'll notice the loopback interface is down. Bringing it up solves the problem.
```
root@ubuntu-xenial:/home/vagrant# ip netns exec netns0 ip link set lo up
```
```
root@ubuntu-xenial:/home/vagrant# ip netns exec netns0 ping localhost
PING localhost (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.048 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.032 ms
^C
--- localhost ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.032/0.040/0.048/0.008 ms
```

# Creating a veth pair
To create a veth pair with devices named `veth-default` and `veth-netns0`:
```
ip link add veth-default type veth peer name veth-netns0
```

`ip link set` allows you to change device attributes. In particular, `netns` will "move the device to the network namespace associated with name NETNSNAME or process PID." In our case, we're going to move one of the veth devices our new namespace:
```
ip link set veth-netns0 netns netns0
```

You can see them listed with `ip a` and `ip netns exec netns0 ip a`.

# Assigning IPs to our veth devices
Before we can `ping` our veth pair devices we're going to need to assign IP addresses to those them.

Here we assign private IPs:
```
ip addr add 10.0.3.1/24 dev veth-default
ip netns exec netns0 ip addr add 10.0.3.2/24 dev veth-netns0
```

# ping veth pair
then we bring them up:
```
ip link set veth-default up
ip netns exec netns0 ip link set veth-netns0 up
```

and we can ping both ways:
```
root@ubuntu-xenial:/home/vagrant# ping 10.0.3.2
PING 10.0.3.2 (10.0.3.2) 56(84) bytes of data.
64 bytes from 10.0.3.2: icmp_seq=1 ttl=64 time=0.052 ms
64 bytes from 10.0.3.2: icmp_seq=2 ttl=64 time=0.043 ms
64 bytes from 10.0.3.2: icmp_seq=3 ttl=64 time=0.043 ms
^C
--- 10.0.3.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.043/0.046/0.052/0.004 ms
```
```
root@ubuntu-xenial:/home/vagrant# ip netns exec netns0 ping 10.0.3.1
PING 10.0.3.1 (10.0.3.1) 56(84) bytes of data.
64 bytes from 10.0.3.1: icmp_seq=1 ttl=64 time=0.056 ms
64 bytes from 10.0.3.1: icmp_seq=2 ttl=64 time=0.046 ms
64 bytes from 10.0.3.1: icmp_seq=3 ttl=64 time=0.042 ms
^C
--- 10.0.3.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 0.042/0.048/0.056/0.005 ms
```

<small>Note: this doesn't work if they are on different subnets.</small>

One step closer to pinging the Internet.

# NAT and packet forwarding

I'm going to use NAT and packet forwarding to connect to the Internet from `netns0`. An alternative would've been to create a network bridge including `veth-netns0` and another Internet-connected device in the default namespace (edit: I think I still need to set up NAT if I want to talk to the Internet).

This is my first time _actually_ setting up a NAT!

## Enabling forwarding
You need to enable and it `echo 1 > /proc/sys/net/ipv4/ip_forward` and you can check its value afterwards `sysctl -a | grep ip_forward`.

## Packet forwarding
Forward packets between `veth-default` and the Internet-connected device `enp0s3`.
```
iptables -A FORWARD -o enp0s3 -i veth-default -j ACCEPT
iptables -A FORWARD -i enp0s3 -o veth-default -j ACCEPT
```

You can view them with `iptables -L -v`.

## IP Masquerading
And setting up the NAT to translate IP on packets from `veth-netns0` to the IP of `enp0s3` as they are about to go out (`POSTROUTING`):
```
iptables -t nat -A POSTROUTING -s 10.0.3.2/24 -o enp0s3 -j MASQUERADE
```

Should be okay? I'm missing one thing.

## Default gateway
If you check out the route tables with `route` within the namespace you'll see there is no gateway!
```
root@ubuntu-xenial:/home/vagrant# ip netns exec netns0 route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.3.0        *               255.255.255.0   U     0      0        0 veth-netns0
```

Inside our network namespace we want to use `veth-default` as our gateway which will then forward our packets.
```
ip netns exec netns0 ip route add default via 10.0.3.1
```

# This, Jen, is the Internet
And now we can `ping` example.com from inside our network namespace:
```
root@ubuntu-xenial:/home/vagrant# ip netns exec netns0 ping example.com
PING example.com (93.184.216.34) 56(84) bytes of data.
64 bytes from 93.184.216.34: icmp_seq=1 ttl=61 time=23.0 ms
64 bytes from 93.184.216.34: icmp_seq=2 ttl=61 time=23.6 ms
64 bytes from 93.184.216.34: icmp_seq=3 ttl=61 time=55.9 ms
64 bytes from 93.184.216.34: icmp_seq=4 ttl=61 time=23.8 ms
64 bytes from 93.184.216.34: icmp_seq=5 ttl=61 time=24.0 ms
^C
--- example.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13068ms
rtt min/avg/max/mdev = 23.020/30.111/55.929/12.914 ms
```

