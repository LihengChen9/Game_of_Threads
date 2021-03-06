# I modified this POC that come from following git repository 

Original git repo is here (I modify it for making it work well):

- https://github.com/jose-sv/hogwild_pytorch

# Overview

`nukemod` is a kernel module that can be used to perform controlled-channel attacks and to halt / resume user-space threads from the operating system. It is used in the paper:

- _"Game of Threads: Enabling Asynchronous Poisoning Attacks"_ (__ASPLOS 2020__)

In particular, this repository contains the kernel code used in the evaluation of the attack against our SGX proof-of-concept (cf. Section 6 in the paper).
The full code artifact of the paper is available at:

- https://github.com/jose-sv/hogwild_pytorch

# Supported Hardware
We tested this code on a bare-metal machine with an Intel i7-6700K CPU @ 4.00GHz.
We cannot guarantee that it works on other CPUs or in virtualized environments.

# Required Setup
- Ubuntu 18.04 LTS

# Prerequisites
To monitor page-faults, `nukemod` hooks the page fault handler of the Linux kernel.
However, this is not allowed by default in the Linux kernel.
To circumvent this limitation, we minimally modified kernel 5.4 so that it allows to hook the page fault handler.
Here are the instructions to patch and install this kernel.

- Install the required packages by running `sudo apt install -y build-essential ocaml automake autoconf libtool wget python libssl-dev bc`.
- Download Ubuntu kernel 5.4 from offcial site:
  - https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.4.tar.gz
- Extract the downloaded kernel into a directory `linux-5.4`.
- Patch the extracted kernel using our provided kernel patch `kernel_5.4.patch`(`kernel_5.4.patch.back`,  follows the author's work, remove `NOKPROBE_SYMBOL` label of `do_page_fault`,` __do_page_fault`, `do_user_addr_fault`, but I think there is no need). To do this, `cd` into the directory `linux-5.4` and run `patch -p1 < ../kernel_5.4.patch`.
- Compile and install the patched kernel. Instructions for this step are available in the README of the kernel itself. In short, you can run:
```sh
cp /boot/config-`uname -r` .config (or make menuconfig)
make -j `nproc` && sudo make modules_install && sudo make install
```
- After installing the custom kernel, make sure to add the kernel boot parameters `nosmap` and `transparent_hugepage=never` to grub (`noacpi` may helpful).
This can be done by modifying a line in the file `/etc/default/grub`:
```sh
GRUB_CMDLINE_LINUX_DEFAULT="nosmap transparent_hugepage=never"
```
- Run `sudo update-grub` to apply the edits to the configuration.
- Reboot your machine into the custom kernel with the custom configuration.

# Usage
(I merge following cmd into load_nuke.sh)
- Compile `nukemod` module by running `make`.
- (optional) Clear the message buffer of the kernel using `sudo dmesg --clear`.
- Create a device file for `nukemod` using `sudo mknod /dev/nuke_channel c 511 0`.
- Load `nukemod` using `sudo insmod nuke.ko`. You can check if it loaded correctly by running `dmesg`.
- Now you can launch the user-space APA attack from this repo: https://github.com/jose-sv/sgx_scheduling.
The user-space attack code will invoke the functions that are in this module.
- (optional) You can see what the kernel module was doing during the attack using `dmesg`.
- When you are done with the attack, unload the kernel module using `sudo rmmod nuke`.

# Credits
Some of this code is inspired from other repositories:

- https://github.com/heartever/SPMattack
- https://github.com/dskarlatos/MicroScope
