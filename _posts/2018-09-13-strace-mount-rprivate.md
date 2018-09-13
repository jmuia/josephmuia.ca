---
layout: post
title: strace mount --rprivate /
---

Sometimes I have trouble finding the information I'm looking for in the Linux man pages.

Recently, I wanted to do the equivalent of the following `mount(8)` command in Go using the `mount(2)` syscall:
```
mount --make-rprivate /
```

The signature of `mount(2)` looks like this:
```
int mount(const char *source, const char *target,
          const char *filesystemtype, unsigned long mountflags,
          const void *data);
```

`--make-rprivate` will translate to `mountflags` and `/` is the `target`.

But I wasn't sure if the `mount --make-rprivate /` command would make some assumptions or gather information about the current state of the mount to fill in the other fields.

`man 8 mount` didn't seem to say.


So, I just used `strace` to see for myself:
```
$ strace mount --make-rprivate /
...
mount("none", "/", NULL, MS_REC|MS_PRIVATE, NULL) = 0
...
```
And there it was, dummy/null values for `source`, `filesystemtype`, and `data`.

Later, with a fresh pair of eyes and a cup of coffee, I read the `mount(2)` man pages:
```
Changing the propagation type of an existing mount
    ...

    The source, filesystemtype, and data arguments are ignored.
```

This makes sense; the command changes the propagation type, so what purpose would the other arguments have?

