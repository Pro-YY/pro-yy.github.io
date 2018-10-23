---
layout: post
title: "building linux kernel module"
date: 2018-10-23 10:30:28 +0800
comments: true
categories: 
---


## Overview
- What is Linux loadable kernel module(LKM)?

A loadable kernel module (LKM) is a machanism for adding/removing code from Linux kernel **at run time**.
Many of device drivers are implemented through this way, otherwise the monolithic kernel would be too large.

LKM communicates with user-space applications through system calls, and it can access almost all the objects/services of the kernel.
LKM can be inserted to the monolithic kernel at any time -- usually at booting or running phase.

Writing LKM has many advantages against directly tweaking the whole kernel. For LKM can be dynamically inserted or removed at run time, we don't need to recomplie the whole kernel nor reboot, and it's more shippable.

So, the easiest way to start kernel programming is to write a module - a piece of code that can be dynamically loaded into the kernel.

- How is the LKM different from an user-space application?

LKM is run in kernel space, which is quite different.

First off, the code is always asynchronous, which means it doesn't execute sequetially and may be interrupted at any time. Thus programmers should always care about the concurrency as well as reentrant issues. Unlike user-space application, which has an entry-point like `main()` and then execute and exit, the LKM is more like a complicated event-driven server that internally has the ability to interract with various kernel services, and externally provides system calls as its user-space `api`. 

Secondly, there's only a fixed and small stack, resource cleanup as well as utilization should always be highly considered. While as for the user-space application, the resource quota is fairly sufficient.

Thirdly, note that there's no floating-point math.


## Prepare Headers
```sh
apt search linux-headers-$(uname -r)
# get our kernel release: 4.18.0.kali2-amd64
apt install linux-headers-4.18.0.kali2-amd64
```

## Simple Module Code

*hello.c*
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Brooke Yang");
MODULE_DESCRIPTION("A simple Linux moadable mernel module");
MODULE_VERSION("0.1");

static char *name = "world";
module_param(name, charp, S_IRUGO);
MODULE_PARM_DESC(name, "The name to display");

static int __init hello_init(void) {
	pr_info("HELLO: Hello, %s!\n", name);
	return 0;
}

static void __exit hello_exit(void) {
	pr_info("HELLO: Bye-bye, %s!\n", name);
}

module_init(hello_init);
module_exit(hello_exit);
```

**pr_info** is a more convenient way of debugging, comparing to the old-style **printk**.
```sh
printk(KERN_INFO ...);  
```

*Makefile*
```makefile
obj-m+=hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
```

## Build && Install
Now we can `make` our **hello** module and then a *hello.ko* emerged successfully.
```sh
root@kali:/opt/kernel-modules/hello# make 
make -C /lib/modules/4.18.0-kali2-amd64/build/ M=/opt/kernel-modules/hello modules
make[1]: Entering directory '/usr/src/linux-headers-4.18.0-kali2-amd64'
  CC [M]  /opt/kernel-modules/hello/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /opt/kernel-modules/hello/hello.mod.o
  LD [M]  /opt/kernel-modules/hello/hello.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.18.0-kali2-amd64'
root@kali:/opt/kernel-modules/hello# ls -l
total 556
-rw-r--r-- 1 root root    566 Oct 23 09:41 hello.c
-rw-r--r-- 1 root root 272720 Oct 23 09:41 hello.ko
-rw-r--r-- 1 root root    872 Oct 23 09:41 hello.mod.c
-rw-r--r-- 1 root root 136376 Oct 23 09:41 hello.mod.o
-rw-r--r-- 1 root root 137864 Oct 23 09:41 hello.o
-rw-r--r-- 1 root root    154 Oct 23 09:38 Makefile
-rw-r--r-- 1 root root     42 Oct 23 09:41 modules.order
-rw-r--r-- 1 root root      0 Oct 23 09:41 Module.symvers
```

display module info
```
root@kali:/opt/kernel-modules/hello# modinfo hello.ko
filename:       /opt/kernel-modules/hello/hello.ko
version:        0.1
description:    A simple Linux moadable mernel module
author:         Brooke Yang
license:        GPL
srcversion:     440743A20C6C4688E185D30
depends:        
retpoline:      Y
name:           hello
vermagic:       4.18.0-kali2-amd64 SMP mod_unload modversions 
parm:           name:The name to display (charp)
```


```
root@kali:/opt/kernel-modules/hello# insmod hello.ko
root@kali:/opt/kernel-modules/hello# rmmod hello
root@kali:/opt/kernel-modules/hello# insmod hello.ko name=Brooke
root@kali:/opt/kernel-modules/hello# rmmod hello
```


we can watch the log by `tail -f` the */var/log/kern.log* or just `dmesg`
```
Oct 23 09:51:37 kali kernel: [ 2651.831228] HELLO: Hello, world!
Oct 23 09:51:44 kali kernel: [ 2658.680087] HELLO: Bye-bye, world!
Oct 23 09:51:57 kali kernel: [ 2672.409216] HELLO: Hello, Brooke!
Oct 23 09:52:02 kali kernel: [ 2677.482181] HELLO: Bye-bye, Brooke!
```
Done!

note: charp parm can even be Chinese.

## Conclusions

With this artical, we managed to complete our first yet very simple Linux loadable kernel module(LKM).

We've got a broad view of how the LKMs work. And we should configure our own kernel modules, build and insert/remove them at runtime, and define/pass custom parameters to them.


## References
- [writing a linux kernel module part 1 introduction](http://derekmolloy.ie/writing-a-linux-kernel-module-part-1-introduction/)
- [how to compile linux kernel modul](https://qnaplus.com/how-to-compile-linux-kernel-module/)
- [linux kernel programming basics create loadable kernel module](https://qnaplus.com/linux-kernel-programming-basics-create-loadable-kernel-module/)
- [be a kernel hacker](https://www.linuxvoice.com/be-a-kernel-hacker/)
