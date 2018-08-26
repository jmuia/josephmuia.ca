---
layout: post
title: insmod hello.ko
---

Historically, josephmuia.ca has been a placeholder for links to my profile on other sites.

However, since I've begun writing a bit it seems customary to add a "Hello, World!" post.

Here it goes, in the form of a super simple Linux kernel module.

# Linux kernel modules
Kernel modules are programs that extend the functionality of the kernel without having to reboot the system; they can be loaded and unloaded on demand.

The idea of writing code to run in the kernel is scary; with great power...

Fortunately, this module won't do much other than log the iconic "Hello, World!" message.

# Code

```
#include <linux/module.h> // Required by all modules.
#include <linux/kernel.h> // Required for kernel logging macros.
#include <linux/init.h> // Required for __init and __exit macros.

static int __init hello(void) {
  printk(KERN_INFO "Hello, World!\n");
  return 0;
}

static void __exit goodbye(void) {
  printk(KERN_INFO "Goodbye, cruel World!\n");
}

module_init(hello);
module_exit(goodbye);
```

The code above is cribbed from [this guide](https://www.tldp.org/LDP/lkmpg/2.6/html/index.html).

A kernel module has two functions:
1. "init" which runs when the module is installed
2. "exit" which runs when the module is removed

# Compiling

The below `Makefile` has targets to compile the module, producing `hello.ko`. Just run `make`.

```
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean 

```

I compiled and tested this module using kernel release 4.15.0-30-generic (`uname -r`).

# Installing

Installing is as easy as `sudo insmod ./hello.ko`.

`lsmod` will read the contents of `/proc/modules`, displaying the currently loaded modules. If you're following along you will see `hello` in the list after installing it. 

Better yet, we can view our "Hello, World!" message in the kernel ring buffer with `dmesg`:
```
[ 3309.530533] Hello, World!
```

# Removing

It's been a slice, but it's time to say goodbye. We can remove the module with `sudo rmmod hello`.

Again, we can see the output of `printk()` with `dmesg`:
```
[ 3319.331082] Goodbye, cruel World!
```

