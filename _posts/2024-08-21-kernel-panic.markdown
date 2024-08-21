---
layout: default
title: Oops! My first Kernel Panic
---


# Oops! My first Kernel Panic

## Context

For the first time I was trying out a release candidate. I am using an `Ubuntu 22.04` with a `6.5.0-35-generic` kernel. The `6.11.0-rc1+` had just been released and I decided to download and build the kernel file. I followed Kaiwan M Billimoria's book The Linux Kernel Programming and The Linux Foundation course LFD103 (A Beginner's guide to Linux Kernel Development) as a guide line for building and installing the Kernel.
I copied my existing Kernel config of `6.5.0-35-generic` from `/boot` into the kernel directory as `.config` and performed the `make` and `make modules_install`. I also performed the `update-grub` command to add the new Kernel to the Grub menu.

It initially booted into the system without showing much problems and I tested out some features like the WiFi, running a browser and some more. I even wrote some basic Linux Kernel Modules and tried loading them and that too worked fine. But after shutting down my PC for the time being and switching it on after some time I was greeted with the following messages:

- `error: failure reading sector 0x16f8d40 from 'hd0'.`
- `/dev/root: Can't open blockdev`
- `Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0, 0)`

<!-- ![Image]({{ site.baseurl }}/assets/images/2024-08-21-kernel-panic/panic-image.jpeg) -->

<img src="{{ site.baseurl }}/assets/images/2024-08-21-kernel-panic/panic-image.jpeg" alt="drawing" width="60%"/>

And I was in face to face with the mighty Kernel panic.

## How the boot process works

When a Linux system boot process starts after the Master Boot Record (MBR) step, GRUB is loaded. The kernel needs to be loaded into RAM to start the OS, but the kernel is situated on the hard disk (`/boot/vmlinuz`), and the hard disk is not yet mounted on `/`. Without mounting, no files can be accessed, even the kernel. To overcome this, first `initramfs` (`initrd`) loads in RAM directly and mounts the `/boot` partition in read-only mode. Next, it mounts the hard disk on the `/` partition, and the process continues.

## Trying to come up with a solution

After searching the internet for sometime and reading about different things it was evident that the problem was with the `initramfs`(`initrd.img-6.11.0-rc1+` in my case). So, I started off by booting into the `rescue` of my `6.11.0-rc1+`. But I was faced with the same issue. Upon failure I again booted up into the `6.5.0-35-generic` kernel and tried running the following command to once again generate the `initrd` image

```bash
sudo update-initramfs -c -k 6.11.0-rc1+
```

But I was constantly facing with the error `No space left on device`. Although I had about 7 GB of free space in my `root` partition, the generation of the `initrd` was continuously failing saying the same. Closer inspection of what is being happening led to the understanding that the `initrd` file of `6.11.0-rc1+` is growing high in size while the `initrd.img-6.5.0-35-generic` had around only 65 MB of size.

## Experiment 1

What I first did try out is to strip down the kernel by recompiling the kernel by excluding all the new drivers which where not mentioned in the `/boot/config-6.5.0-35-generic` (My old kernel configuration which is being copied to the kernel source directory as `.config`). 

So, I removed the old kernel and recompiled it.

```bash
sudo rm /boot/config-6.11.0-rc1+
sudo rm /boot/initrd.img-6.11.0-rc1+
sudo rm /boot/System.map-6.11.0-rc1+
sudo rm /boot/vmlinuz-6.11.0-rc1+
sudo rm -rf /lib/modules/6.11.0-rc1+
```

```bash
cd ~/Kernels/linux # Where my kernel source directory is
make clean
make -j16 all
su -c 'make modules_install'
```

After compilation during the `make install` the same error `No space left on device` occurred and it was a big failure.

So, I did the following as I saw in a stack exchange thread that it would drastically reduce the size of the kernel by stripping down the kernel modules

```bash
cd /lib/modules/6.11.0-rc1+
find . -name *.ko -exec strip --strip-unneeded {} +
```

After moving to the `/lib/modules/6.11.0-rc1+` directory is that we use the `find` to list out all object files ending with the name `.ko` and pass these files to the `strip` command (used to discard symbols and other data from object files) along with the --strip-unneeded flag to remove all symbols that are not needed for relocation processing.

**Relocation** is the process of assigning load addresses for position-dependent code and data of a program and adjusting the code and data to reflect the assigned addresses. A linker usually performs **relocation** in conjunction with **symbol resolution**, the process of searching files and libraries to replace symbolic references or names of libraries with actual usable addresses in memory before running a program.

Generating the `initrd` post this step led to yet another failure.
## Experiment 2

Upon going through different articles and stack overflow threads, I got stumbled upon a flag that can be added along with `make modules_install`. There is this flag called `INSTALL_MOD_PATH` where it's definition in the documentation goes as follows


`INSTALL_MOD_STRIP`, if defined will cause modules to be stripped after they are installed. If `INSTALL_MOD_STRIP` is '1', then the default option `--strip-debug` will be used. Otherwise, `INSTALL_MOD_STRIP` value will be used as the options to the strip command.


From this what I am able to understand is running

```bash
su -c 'make INSTALL_MOD_STRIP=1 modules_install' 
```

should be analogous to (while running from the kernel source directory)

```bash
su -c 'make modules_install'
find /lib/modules/6.11.0-rc1+ -name *.ko -exec strip --strip-debug {} +
```

`--strip-debug` strips away debugging symbols.

So, by doing the following I was able to fix my issue

```bash
cd ~/Kernels/linux/ # 6.11.0-rc1+ kernel source directory
su -c 'make INSTALL_MOD_STRIP=1 modules_install make'
```

This time the `initrd` generated successfully and I was able to successfully boot into the new kernel.

## Conclusion

This is not like a standard article that you will find in the internet. This article is widely incomplete due to the following reasons

1. My lack of knowledge on the topic
2. My lack of expertise in writing articles

But the purpose of this article was is to start documenting my Kernel development journey hoping that this will eventually become a good reference for me and for others in the future.
