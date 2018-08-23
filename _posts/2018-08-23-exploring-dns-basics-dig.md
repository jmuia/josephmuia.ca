---
layout: post
title: Exploring DNS basics with dig
---


In this post I'm going to provide an overview of DNS and explore some of the basics using the `dig` tool.

* TOC
{:toc}

# Overview: what is DNS?
DNS (Domain Name System) is a hierarchical, distributed naming system for computers and services.

### It is a naming system
It maps human-friendly domain names (eg. `www.josephmuia.ca`) to computer-friendly Internet addresses (eg. `192.168.0.6`). It is like a phone book for the Internet: you know the name of the computer and DNS will give you the number to call.

### It is hierarchical
A domain name is organized in a hierarchy. `www.josephmuia.ca.` is organized like so: `.` is the root (this is often implied), `ca` is the top-level domain, then `josephmuia`, then `www`.

### It is distributed
In the design of DNS it was decided that many organizations and computers will cooperate in providing this service.

# DNS components

DNS has three major components:
1. **Resource Records**: these store information about the resources associated with a particular domain name, including the type, class, TTL, and the data. A common record is the type `A`, with class `IN` for Internet, and an IP address for the data.

2. **Name Servers**: the servers that have information about subtrees of the domain hierarchy in the form of zone files which contain resource records. A name server may be **authoritative**, holding the authoritative records for a domain. They may be **recursive**, querying other name servers until receiving an answer from an authoritative name server. Name servers cache records (respecting the TTL) to short-circuit future lookups.

3. **Resolvers**: the programs that gather information from name servers to answer a client request. Your operating system might start the process by contacting your ISP's name server, but often your ISP will take over as the resolver (often called a **resolving** name server) and perform a recursive lookup.

# DNS process
Now that we are familiar with the main components involved, we can show how they work together to service a DNS request.

![Diagram of DNS lookup process](/img/dns-domain-lookup.svg)

1. In the diagram, a browser is asking for the address of `josephmuia.ca`. It will first check it's cache before asking the operating system. The operating system will also check it's cache before asking the resolving name server.

2. If the resolving name server doesn't have the record cached, it asks the **root name server**. It is the authority on Top-Level Domain servers. It doesn't know where `josephmuia.ca` lives, but it can tell us the IP address of the name server for `.ca` so we can ask there.

3. The resolving name server contacts the TLD server found in the response from the root. The **Top-Level Domain name servers** have information about TLDs like `.edu`, `.ca`, and `.ninja`. It _also_ doesn't know where `josephmuia.ca` lives, but it can tell us the IP address for the name server that does.

4. The resolving name server then uses this information to contact the name server that has the authoritative information about `josephmuia.ca`.

5. The resolving name server responds to the initial request with the IP address provided by the authoritative name server.

6. Voilà! The browser now has the information needed to contact `josephmuia.ca`.

The TLD NS doesn't always point directly to the authoritative name server for the domain name in question. Take, for example, `networking.cs.example.com`, where `example.com` only knows that `cs.example.com` has information about that subdomain.
 
# An example with dig

Now that we understand how a DNS lookup works from a high-level, let's get our hands dirty by walking through the process on a real domain. We'll use a utility called `dig`, which is useful for performing DNS queries and displaying the answers.

## Basic dig usage

Try `$ dig www.example.com`. You'll see output similar to this:
```
; <<>> DiG 9.10.6 <<>> www.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15843
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.example.com.		IN	A

;; ANSWER SECTION:
www.example.com.	16604	IN	A	93.184.216.34

;; Query time: 3 msec
;; SERVER: 192.168.2.1#53(192.168.2.1)
;; WHEN: Wed Aug 22 21:04:48 EDT 2018
;; MSG SIZE  rcvd: 49
```

The flag `qr` indicates our message was a query (opposed to an answer). `rd` indicates that recursion was desired and `ra` that it was available. The question section reveals our query; the `dig` default is Internet `A` records. The answer section contains the IP address for `example.com`.

## Manually resolving www.example.com

### Finding the root servers
We're going to bypass the resolving name server and go directly to the root. Plain `$ dig` will list the Name Server records (`NS`) for the root.

```
; <<>> DiG 9.10.6 <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21520
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;.				IN	NS

;; ANSWER SECTION:
.			316171	IN	NS	a.root-servers.net.
.			316171	IN	NS	j.root-servers.net.
.			316171	IN	NS	h.root-servers.net.
.			316171	IN	NS	e.root-servers.net.
.			316171	IN	NS	d.root-servers.net.
.			316171	IN	NS	c.root-servers.net.
.			316171	IN	NS	f.root-servers.net.
.			316171	IN	NS	l.root-servers.net.
.			316171	IN	NS	k.root-servers.net.
.			316171	IN	NS	g.root-servers.net.
.			316171	IN	NS	b.root-servers.net.
.			316171	IN	NS	i.root-servers.net.
.			316171	IN	NS	m.root-servers.net.
```

### Asking the root servers for answers
Now let's ask one of the root servers for information about `www.example.com`. `dig` allows you to specify the name server to query in the format `@server`. This can be a name or an IP address. DNS resolver configurations already have the IP addresses of the root name servers, so we'll just use the name here. We'll also use the `+norecurse` option to avoid recursion.

```
$ dig @a.root-servers.net. www.example.com +norecurse
```
```
; <<>> DiG 9.10.6 <<>> @a.root-servers.net. www.example.com +norecurse
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5279
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.example.com.		IN	A

;; AUTHORITY SECTION:
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.

;; ADDITIONAL SECTION:
e.gtld-servers.net.	172800	IN	A	192.12.94.30
b.gtld-servers.net.	172800	IN	A	192.33.14.30
j.gtld-servers.net.	172800	IN	A	192.48.79.30
m.gtld-servers.net.	172800	IN	A	192.55.83.30
i.gtld-servers.net.	172800	IN	A	192.43.172.30
f.gtld-servers.net.	172800	IN	A	192.35.51.30
a.gtld-servers.net.	172800	IN	A	192.5.6.30
g.gtld-servers.net.	172800	IN	A	192.42.93.30
h.gtld-servers.net.	172800	IN	A	192.54.112.30
l.gtld-servers.net.	172800	IN	A	192.41.162.30
k.gtld-servers.net.	172800	IN	A	192.52.178.30
c.gtld-servers.net.	172800	IN	A	192.26.92.30
d.gtld-servers.net.	172800	IN	A	192.31.80.30
```

You'll see numerous authoritative answers for the TLD name servers for `.com`. In the additional section "glue records" are provided so we know the IP address of those name servers.

### Asking the TLD servers for answers
For readability we'll use the name in the `dig` command, but we could also use the IP address (`dig` will do this for us if we provide the name).

```
$ dig @e.gtld-servers.net. www.example.com +norecurse
```
```
; <<>> DiG 9.10.6 <<>> @e.gtld-servers.net. www.example.com +norecurse
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60827
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 2, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.example.com.		IN	A

;; AUTHORITY SECTION:
example.com.		172800	IN	NS	a.iana-servers.net.
example.com.		172800	IN	NS	b.iana-servers.net.

;; ADDITIONAL SECTION:
a.iana-servers.net.	172800	IN	A	199.43.135.53
b.iana-servers.net.	172800	IN	A	199.43.133.53
```

### Asking the authoritative servers for answers
```
$ dig @a.iana-servers.net. www.example.com +norecurse
```
```
; <<>> DiG 9.10.6 <<>> @a.iana-servers.net. www.example.com +norecurse
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15864
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.example.com.		IN	A

;; ANSWER SECTION:
www.example.com.	86400	IN	A	93.184.216.34
```

And we're done! The authoritative name server provided the IP address for `www.example.com`.

### +trace
We didn't have to do all this manually; `dig` provides an option `+trace` that perform an iterative lookup for us.

# Parting thoughts
DNS is neat and `dig` is a powerful tool for exploring it.

If you'd like to learn more about DNS check out the RFCs [1034](https://www.ietf.org/rfc/rfc1034.txt) [1035](https://www.ietf.org/rfc/rfc1035.txt).
`dig` can do much more than demonstrated here, so check out the man pages to learn more about that.

