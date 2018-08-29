---
layout: post
title: strace bash startup files
---

I've seen lots about `strace` on the Internet (erm, [jvns.ca](https://jvns.ca/categories/strace/)). I decided to `strace` some things.

I'd previously attempted to remap caps lock to escape on Ubuntu. I added `setxkbmap -option caps:swapescape` to `~/.profile`. At the time I `source`d the file with the expectation that future logins would also apply the mapping.

However, I noticed that after a restart it was back to caps lock :(

According to `~/.profile` "This file is not read by bash(1), if ~/.bash\_profile or ~/.bash\_login exists."

Neither of those files exist.

I've got a new tool! I can `strace` bash to see if it's actually reading `~/.profile` for configuration.

```
jmuia@jmuia:~$ strace -f -e open bash
```

I saw some familiar startup files, like `.bashrc`, but no `.profile`.

So I've confirmed it wasn't reading `.profile`.

Turns out I need to learn a bit about login shells (which was actually mentioned in `.profile`, oops).

