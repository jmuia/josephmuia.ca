---
layout: post
title: How does the docker run -p option work?
---

According to the `docker run --help`:
```
  -p, --publish list                   Publish a container's port(s) to the host
```

How does that actually work? `iptables`!

I initially read about this [here](https://fralef.me/docker-and-iptables.html), though it's missing a note about the nat table.

If no containers are running, there's nothing interesting in the DOCKER filter and nat chains:
```
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

$ sudo iptables -L DOCKER
Chain DOCKER (1 references)
target     prot opt source               destination

$ sudo iptables -t nat -L DOCKER
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
```

However, when we map a host port to a container port, we see the corresponding rules in `iptables`:

```
$ docker run -d -p 9090:80 nginx
49734779e1486cf27b7a6dba7118d9ecf19ea4ea6f716da4e766c4a258e6af36

$ docker port 49734779e1486cf27b7a6dba7118d9ecf19ea4ea6f716da4e766c4a258e6af36
80/tcp -> 0.0.0.0:9090

$ sudo iptables -L DOCKER
Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             172.17.0.2           tcp dpt:http

$ sudo iptables -t nat -L DOCKER
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
DNAT       tcp  --  anywhere             anywhere             tcp dpt:9090 to:172.17.0.2:80
```



