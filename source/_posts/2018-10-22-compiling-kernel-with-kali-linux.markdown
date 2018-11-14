---
layout: post
title: "compiling kernel with kali linux"
date: 2018-10-22 09:57:10 +0800
comments: true
categories: Linux
---

With the instructions in [Recompiling the Kali Linux Kernel](https://docs.kali.org/development/recompiling-the-kali-linux-kernel), we can recompile the whole linux kernel of our kali linux.


## Install Dependencies
We need to start by installing several build dependency packages, some of them may have already been in installed.
```
apt install build-essential libncurses5-dev fakeroot bison libssl-dev libelf-dev
```

## Download the Kernel Source
For my system, the kernel version is 4.18, which will be used in following example. Of course, the workflow of other version is just the same.
```
apt install linux-source-4.18
[...]
ls /usr/src
linux-config-4.18 linux-patch-4.18-rt.patch.xz linux-source-4.18.tar.xz
```
Then we get the compressed archive of the kernel sources, and we'll extract these files in our working directory, (no special permission need for compiling the kernel). In our example, we use */opt/kernel*, and the *~/kernel* is also an appropriate place.

```
mkdir /opt/kernel; cd /opt/kernel
tar -xaf /usr/src/linux-source-4.18.tar.xz
```
Optionally, we may also apply the *rt* patch, which is for real-time os features.
```
cd /opt/kernel/linux-source-4.18
xzcat /usr/src/linux-patch-4.18-rt.patch.xz | patch -p1
```

## Configure the Kernel
When building a more recent version of kernel (possibly with an specific patch), the configuration should at first be kept as close as possible to the current running kernel, shown by **uname -r**. It is sufficient to just copy the currently-used kernel config to the source directory.
```
cd /opt/kernel/linux-source-4.18
cp /boot/config-4.18.0-kali2-amd64 .config
```

If you need to make some changes or decide to reconfigure all things from scratch, just call **make menuconfig** command and inspect all the details.
Note: we can tweak a lot in this phase.

## Write Some Code
Add one line of code for test(fun), in file *init/main.c*, **start_kernel** function
```
	pr_notice("Brookes's customized kernel starting: %s %d\n", __FUNCTION__, __LINE__);
	pr_notice("%s", linux_banner);
```

## Build the Kernel
Once configured, we can **make** the kernel. Rather than invoking **make deb-pkg** as the official doc suggested, we use **make bindeb-pkg** here, which will not generate Debian source package, or invoke **make clean**.
```
time make -j4 bindeb-pkg LOCALVERSION=-custom KDEB_PKGVERSION=$(make kernelversion)-$(date +%Y%m%d)
```
After a while, we get following package in the parent directory
```
linux-headers-4.18.10-custom_4.18.10-20181021_amd64.deb   # headers
linux-image-4.18.10-custom_4.18.10-20181021_amd64.deb     # kernel image
linux-image-4.18.10-custom-dbg_4.18.10-20181021_amd64.deb # kernel image with debugging symbols
linux-libc-dev_4.18.10-20181021_amd64.deb                 # headers of user-space library
```

### Trouble Shooting
```
No rule to make target 'debian/certs/test-signing-certs.pem', needed by 'certs/x509_certificate_list'. Stop
```
Solve: comment/delete the corresponding config line.

## Install/Remove the Kernel

```
dpkg -i ../linux-image-4.18.10-custom_4.18.10-custom_4.18.10-20181021_amd64.deb
reboot
```
Once booted, we can use **dmesg** to verify our printk message.
Removing kernel can also be done with dpkg.
```
dpkg -r linux-image-4.18.10-custom
```
