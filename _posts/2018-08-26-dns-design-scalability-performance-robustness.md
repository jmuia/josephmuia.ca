---
layout: post
title: "DNS Design: Scalability, Performance, Robustness"
---

As explained in *[Exploring DNS basics with dig]({% post_url 2018-08-23-exploring-dns-basics-dig %})*, DNS is a conceptually simple system. However, it's design has properties that make it a scalable, performant, and robust system.

* TOC
{:toc}

# Motivations for DNS
Section 2.1 of [RFC 1034](https://tools.ietf.org/html/rfc1034) describes the motivations for developing DNS, which are rooted in the growth of the Internet. It was unscalable for a single organization to host the all the mappings, both from a bandwidth perspective and a management perspective. In addition, the growing sophistication of applications demanded a more general purpose naming service.

# DNS design

## Layer of indirection
Naming adds a layer of indirection to a system. This may be obvious, but it's worth pointing out.
As the [fundamental theorem of software engineering](https://en.wikipedia.org/wiki/Fundamental_theorem_of_software_engineering) remarks, "we can solve any problem by introducing an extra level of indirection."

Indirection helps manage complexity in computer systems and makes them more robust. It decouples components of the system and supports their substitution.

The primary design goal of DNS was to provide a consistent name space for referring to resources, inherently providing a layer of indirection to its users.

DNS provides additional means to add indirection via `CNAME` records, which are an alias to the canonical name of the resource.

## Distributed
Following the motivations for DNS, the growing size of the database and frequency of updates meant that the system needed to be distributed in order to perform at scale.

It was distributed in a few ways. The management of subdomains are distributed to numerous organizations. This means that these organizations are responsible for the hardware servicing queries as well as the administration of the domains themselves. DNS's referral architecture means that a client can ask any name server for an answer rather than always relying on the root name servers. This is particularly useful for requests within a domain as it may never leave an organization's network.

## Caching and eventual consistency
Eventual consistency is useful in systems where performance and availability are a high priority and temporary inconsistency is tolerable.
In the design of DNS it was assumed that a) most of the data in the system will change very slowly, and b) access to information is more critical than instantaneous updates or guarantees of consistency.

As such, caching is an important contributor to the performance of DNS.
Name servers cache records, with respect to the TTL determined by the authority, speeding up future resolution requests substantially.

(*Try it out with `dig` by resolving an obscure domain and see if a future attempt is faster*.)

While this optimization is useful on all name servers, it's effect is compounded on recursive name servers that will service many requests and cache many parts of the domain hierarchy.

## Redundancy
Redundancy is built into DNS. It's RFCs require that every zone be available on at least two servers.
Although it doesn't seem to be required in the RFC, it is a good idea for these servers to be distributed geographically to avoid broader system failures impacting all replicas.

Redundancy is particularly important higher in the hierarchy. According to [root-servers.org](http://root-servers.org/) there are 937 instances of the 13 root servers running, distributed globally.

Similar to DNS itself, many of the named resources must be redundant. DNS allows a name to be bound to several different IP addresses.

## Transport
Section 4.2 of [RFC 1035](https://tools.ietf.org/html/rfc1035) notes that both reliable and unreliable transport mechanisms are used in DNS.

Reliable mechanisms are required for activities such as zone transfers, but can be used for any activity. Unreliable datagram-based transport is preferred for queries due to their lower overhead and better performance.

DNS is a real-time application and queries don't benefit from the reliability of a protocol like TCP; the connectionless nature of UDP is an asset:
* there's no time spent handshaking
* the server spares resources otherwise required for connection state management
* there is no congestion-control mechanism delaying the transmission of datagrams
* it's fire and forget.
If a datagram doesn't receive a response the client can quickly re-send the datagram or ask another server, whereas TCP would spend more time to achieve a reliable delivery.

## Resolver design
Section 5.3 of [RFC 1034](https://tools.ietf.org/html/rfc1034) provides resources to help resolver designers, including a set of recommended priorities. The top of these is to set bounds for the amount of work done to prevent infinite loops and unintentional cascading requests.

# Parting thoughts
DNS employs some basic design patterns in an intelligent way to build a scalable and performant naming system.

I've referred to the DNS RFCs a lot in this post. I've only read sections of them, but I found the parts I read to be approachable and would encourage using them as a reference to learn more.

Kurose and Ross *[Computer Networking: A Top-Down Approach](https://www.pearson.com/us/higher-education/program/Kurose-Computer-Networking-A-Top-Down-Approach-7th-Edition/PGM1101673.html)* provided the information in the Transport section and Saltzer and Kaashoek *[Principles of Computer System Design](https://www.elsevier.com/books/principles-of-computer-system-design/saltzer/978-0-12-374957-4)* offered general system design principles and their application to DNS throughout the post.

