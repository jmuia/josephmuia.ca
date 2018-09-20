---
layout: post
title: ps, /proc, and the PID namespace
---

PID namespaces isolate the process ID number space. The first process created in a new PID namespace will be PID 1.

```
jmuia@host$ sudo unshare --pid --fork
root@unshare# echo $$
1
```

However, if you ask `ps` it will tell you otherwise.
```
jmuia@host$ sudo unshare --pid --fork
root@unshare# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1 22.7  0.5  38000  6044 ?        Ss   00:20   0:26 /sbin/init
root         2  0.0  0.0      0     0 ?        S    00:20   0:00 [kthreadd]
...
root      2038  0.0  0.0   5996   736 pts/0    S    00:22   0:00 unshare --pid --fork
root      2039  3.0  0.4  21288  5052 pts/0    S    00:22   0:00 -bash
root      2053  0.0  0.3  36056  3236 pts/0    R+   00:22   0:00 ps aux
```

On Linux, the `ps` command works by reading files in `/proc`, so it won't be accurate if you've only entered a new PID namespace. The trick is to enter a new mount namespace and mount a new proc file system (since it inherits the caller's mount list).

An interesting thing, though, is that you can't interact with a host PID even though you can see it in `/proc`.

```
# htop running on host
jmuia@host$ ps aux | grep htop
jmuia    10934  1.4  0.0  27524  4880 pts/1    S+   20:54   0:01 htop

# can still see htop in /proc from the PID namespace
jmuia@host$ sudo unshare --pid --fork
root@unshare# ps aux | grep htop
jmuia    10934  1.4  0.0  27524  4880 pts/1    S+   20:54   0:02 htop

# but we can't kill it, because that PID doesn't exist in our namespace
root@unshare# kill -9 10934
-bash: kill: (10934) - No such process
```
