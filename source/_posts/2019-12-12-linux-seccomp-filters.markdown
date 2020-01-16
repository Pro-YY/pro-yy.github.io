---
layout: post
title: "Linux seccomp filters"
date: 2019-12-12 22:20:45 +0800
comments: true
categories: linux
---

## Overview

Seccomp (short for Secure Computing mode) is a computer security facility in the Linux kernel. It was merged into the Linux kernel mainline in kernel version 2.6.12, which was released on March 8, 2005. Seccomp allows a process to make a one-way transition into a "secure" state in which it cannot make some system calls. If it attempts, the kernel will terminate the process tith SIGSYS. Seccomp-BPF was released in 2012, providing more syscall filtering features on bpf. It is used in many sandbox-like applications (i.e. Chrome/Chromium, Firefox, Docker, QEMU, Android, Systemd, OpenSSH...) for resource isolation purposes.

## Basic example

Question: How to block specified syscalls?

First off, we need header files to use libseccomp2. Get the package installed:
```
apt install libseccomp-dev
```

The following code (function *filter_syscalls()*) shows how we use seccomp in common. It filters the `fchmodat` and `symlinkat` syscalls. And also blocks `write` syscall, if the write count argument exceeds 2048.

```
// ... other headers and macros ommited
#include <seccomp.h>

int filter_syscalls() {
    int ret = -1;
    scmp_filter_ctx ctx;

    log_debug("filtering syscalls...");
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    if (!ctx) { log_error("error seccomp ctx init"); return ret; }

    // prohibits specified syscall
    ret = seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(fchmodat), 0);
    if (ret < 0) { log_error("error seccomp rule add: fchmodat"); goto out; }

    ret = seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(symlinkat), 0);
    if (ret < 0) { log_error("error seccomp rule add: symlinkat"); goto out; }

    // limit syscall arguments
    ret = seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(write), 1,
            SCMP_A2_64(SCMP_CMP_GT, 2048));
    if (ret < 0) { log_error("error seccomp rule add: write"); goto out; }

    ret = seccomp_load(ctx);
    if (ret < 0) { log_error("error seccomp load"); goto out; }

out:
    seccomp_release(ctx);
    if (ret != 0) return -1;

    return 0;
}

extern char **environ;

int main(int argc, char *argv[]) {
    int ret = -1;

    ret = filter_syscalls();
    if (ret != 0) { log_error("filter syscall failed"); return EXIT_FAILURE; }

    char *prog = "/bin/bash";
    ret = execve(prog, (char *[]){prog, 0}, environ);
    log_debug("%d", ret);
    if (ret < 0) { log_error("exec failed"); return EXIT_FAILURE; }

    return EXIT_SUCCESS;
}
```

For all source code & detail, check [here](https://github.com/Pro-YY/seccomp-demo).

Note: compile the above with `-lseccomp` flags, and run it when we get our secured shell.

Then, try it with the execed bash prompt:
```
brooke@VM-250-12-ubuntu:~/seccomp_demo$ gcc seccomp_basic.c -l seccomp && ./a.out
[DEBUG]seccomp_basic.c: 32: filtering syscalls...
brooke@VM-250-12-ubuntu:~/seccomp_demo$ chmod -x a.out	# test fchmodat
Bad system call (core dumped)
brooke@VM-250-12-ubuntu:~/seccomp_demo$ ln -s a.out		# test symlinkat
Bad system call (core dumped)
brooke@VM-250-12-ubuntu:~/seccomp_demo$ echo "hello"		# test write
hello
brooke@VM-250-12-ubuntu:~/seccomp_demo$ cat seccomp_basic.c	# test write
Bad system call (core dumped)

brooke@VM-250-12-ubuntu:~/seccomp_demo$ cat /proc/$$/status
...
NoNewPrivs:     1		# cannot be applied to child processes with greater privileges
Seccomp:        2		# Seccomp filter mode
...

brooke@VM-250-12-ubuntu:~/seccomp_demo$ sudo ls
sudo: effective uid is not 0, is /usr/bin/sudo on a file system with the 'nosuid' option set or an NFS file system without root privileges?
brooke@VM-250-12-ubuntu:~/seccomp_demo$ exit	# Don't forget quit bash
```
As expected, the process (subprocess) invoke filtered syscall get SIGSYS, and core-dumped.

## Export filter's bpf
Underneath, seccomp performs filtering by using bpf, which we'll explain later. The libseccomp provide useful funcitons to generate and output the corresponding bpf as well as pfc (Pseudo Filter Code). Thus we can take a more close look.

For a trival case, we only filter the `fchmodat` syscall, and export bpf:
```
int filter_syscalls() {
    int ret = -1;
    scmp_filter_ctx ctx;

    log_debug("filtering syscalls...");
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    if (!ctx) { log_error("error seccomp ctx init"); return ret; }

    // prohibits specified syscall
    ret = seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(fchmodat), 0);
    if (ret < 0) { log_error("error seccomp rule add: fchmodat"); goto out; }

    ret = seccomp_load(ctx);
    if (ret < 0) { log_error("error seccomp load"); goto out; }


    // export bpf
    int bpf_fd = open("seccomp_filter.bpf", O_CREAT | O_WRONLY | O_TRUNC, 0666);
    if (bpf_fd == -1) { log_error("error open"); goto out; }
    ret = seccomp_export_bpf(ctx, bpf_fd);
    if (ret < 0) { log_error("error export"); goto out; }
    close(bpf_fd);

    // export pfc
    int pfc_fd = open("seccomp_filter.pfc", O_CREAT | O_WRONLY | O_TRUNC, 0666);
    if (pfc_fd == -1) { log_error("error open"); goto out; }
    ret = seccomp_export_pfc(ctx, pfc_fd);
    if (ret < 0) { log_error("error export"); goto out; }
    close(pfc_fd);

out:
    seccomp_release(ctx);
    if (ret != 0) return -1;

    return 0;
}
```

We got 2 files, bpf and pfc:
```
$ hd seccomp_filter.bpf		# hexdump bpf file
00000000  20 00 00 00 04 00 00 00  15 00 00 05 3e 00 00 c0  | ...........>...|
00000010  20 00 00 00 00 00 00 00  35 00 00 01 00 00 00 40  | .......5......@|
00000020  15 00 00 02 ff ff ff ff  15 00 01 00 0c 01 00 00  |................|
00000030  06 00 00 00 00 00 ff 7f  06 00 00 00 00 00 00 00  |................|
00000040
```

```
$ cat seccomp_filter.pfc
#
# pseudo filter code start
#
# filter for arch x86_64 (3221225534)
if ($arch == 3221225534)
  # filter for syscall "fchmodat" (268) [priority: 65535]
    if ($syscall == 268)
        action KILL;
          # default action
            action ALLOW;
# invalid architecture action
action KILL;
#
# pseudo filter code end
#
```

It seems quite straightforward. And there's an awesome tool: [seccomp-tools](https://github.com/david942j/seccomp-toolsj) which can disassembles *seccomp_filter.bpf* above:
```
 line  CODE  JT   JF      K
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x05 0xc000003e  if (A != ARCH_X86_64) goto 0007
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x35 0x00 0x01 0x40000000  if (A < 0x40000000) goto 0005
 0004: 0x15 0x00 0x02 0xffffffff  if (A != 0xffffffff) goto 0007
 0005: 0x15 0x01 0x00 0x0000010c  if (A == fchmodat) goto 0007
 0006: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0007: 0x06 0x00 0x00 0x00000000  return KILL
```

## Seccomp-BPF

Seccomp-BPF is just an extension of cBPF (classical Berkeley Packet Filter, Note: not eBPF).
The **tiny**  bpf program runs on a specific VM in kernel, with a rather limited registers and a more reduced instruction set.

BPF code definitions in */usr/include/linux/filter.h*:
```
struct sock_filter {    /* Filter block */
        __u16   code;   /* Actual filter code */
        __u8    jt;     /* Jump true */
        __u8    jf;     /* Jump false */
        __u32   k;      /* Generic multiuse field */
};
```

We can of course, directly apply seccomp-bpf binary code with prctl(), which wraps the `seccomp` syscall, 
to gain more fine-graind control of our bpf. But in most casses, those libseccomp wrappers, like `seccomp_rule_add()` just works.
The binary code is the same as the just hexdumped file for filtering `fchmodat`.

```
int filter_syscalls() {
    int ret = -1;

    log_debug("filtering syscalls with bpf...");

    struct sock_filter code[] = {
        /* op,   jt,   jf,     k    */
        {0x20, 0x00, 0x00, 0x00000004},
        {0x15, 0x00, 0x05, 0xc000003e},
        {0x20, 0x00, 0x00, 0x00000000},
        {0x35, 0x00, 0x01, 0x40000000},
        {0x15, 0x00, 0x02, 0xffffffff},
        {0x15, 0x01, 0x00, 0x0000010c}, // 268 fchmodat
        {0x06, 0x00, 0x00, 0x7fff0000},
        {0x06, 0x00, 0x00, 0x00000000},
    };

    struct sock_fprog bpf = {
        .len = ARRAY_SIZE(code),
        .filter = code,
    };

    ret = prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    if (ret < 0) { log_error("error prctl set no new privs"); return EXIT_FAILURE; }

    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &bpf);
    if (ret < 0) { log_error("error prctl set seccomp filter"); return EXIT_FAILURE; }

    return 0;
}
```

## Performance Overhead
There is no such thing as a free lunch, so as the seccomp-bpf. After all,it is a hooking program, that runs each time whever and whatever a syscall invoked.
We benchmarked 3 senarios: no filter, filter that blocks 1 syscall, and filter that blocks 100 syscall (a more sophisticated bpf).
And we measured the time elpased during 10million `write()` syscall, and plotted as following:

<div id="hc_test" style="min-width: 310px; height: 400px; margin: 0 auto">hc_test</div>

<script src="https://code.highcharts.com/highcharts.src.js"></script>
<script type="text/javascript">

function draw() {
Highcharts.chart('hc_test', {
    chart: {
      type: 'column'
    },

    title: {
        text: ''//
    },

    subtitle: {
        text: ''//
    },

    xAxis: {
        categories: ['without filter', '1-filter bpf', '100-filter pbf']
    },
    yAxis: {
        title: {
            text: 'time (seconds)',
        },
        max: 5,
        min: 0,
    },

    tooltip: {
        formatter: function () {
            return '<b>' + this.x + '</b><br/>' +
                this.series.name + ': ' + this.y + '<br/>';
        }
    },
plotOptions: {
        bar: {
            dataLabels: {
                enabled: true
            }
        }
    },

    series: [{
        type: 'line',
        name: '256 write real',
        data: [3.991, 4.157, 4.506]
    }, {
        type: 'line',
        name: '2048 write real',
        data: [3.959, 4.162, 4.502]
    }, {
        type: 'line',
        name: '256 write sys',
        data: [2.763, 3.011, 3.397]
    }, {
        type: 'line',
        name: '2048 write sys',
        data: [2.797, 3.059, 3.321]
    }]

});

}

draw();
</script>

As it shows, the overhead is around 5%~10%, and will be even more with the larger bpf code.

## Summary
In this post, we managed to filter syscalls  with several seccomp-related facilities, inspect the seccomp-bpf code, and understand its costs.
This would be helpful especially if you're implementing your sandbox-like applications that need security concerns.
Wish you enjoy hacking!

## References
- [Seccomp(2) Linux mannual page](http://man7.org/linux/man-pages/man2/seccomp.2.html)
- [seccomp_init(3) Linux mannual page](http://man7.org/linux/man-pages/man3/seccomp_init.3.html)
- [Kernel document: seccomp filter](https://www.kernel.org/doc/html/v4.16/userspace-api/seccomp_filter.html)
- [A seccomp overview](https://lwn.net/Articles/656307/)
- [Using seccomp to limit the kernel attack surface](http://man7.org/conf/lpc2015/limiting_kernel_attack_surface_with_seccomp-LPC_2015-Kerrisk.pdf)
- [libseccomp source](https://github.com/seccomp/libseccomp)
- [A seccomp overview](https://lwn.net/Articles/656307/)
- [Linux Socket Filtering aka Berkeley Packet Filter (BPF)](https://www.kernel.org/doc/Documentation/networking/filter.txt)
