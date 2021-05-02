---
layout: post
title: Running Linux in Hyperkit
date: 2020-05-01
---

If you are not familiar with it
[Hyperkit][1] is the hypervisor used in Docker for MacOS,
based on [xhyve][2].

[1]: https://github.com/moby/hyperkit
[2]: https://github.com/machyve/xhyve

Ok, Docker is already running Linux, so we're done, right?
Well Docker containers have a very disposable feel to them.
What if we want something to tinker with and look after?

## Getting Hyperkit
If you have Docker for Mac installed,
then you already have `/usr/local/bin/hyperkit` on hand.
Wonderful.
If not, it's quite simple to build.
```shell
% git clone https://github.com/moby/hyperkit.git
% cd hyperkit
% make
```

Ignore the instructions about opam and ocaml dependendencies for now.
It can get a bit fiddly.
You'll be missing out on the qcow2 support,
but it's not essential.

Hyperkit has support for running Linux via the kexec mechanism.
What that means is, running it looks like
```
hyperkit ... -f kexec,$KERNEL,$INITRD,"$CMDLINE"
```
or more explicitly, something like
```
hyperkit ... -f kexec,vmlinuz,initrd.gz,"earlyprintk=serial console=ttyS0"
```

Here's the first _gotcha_. Most distributions provide a live CD/USB iso.
Where do these `vmlinuz` and `initrd` files come from?

We're going to use Alpine Linux for our example.
Download the x86\_64 ISO virtual from <https://alpinelinux.org/downloads/>
e.g.
```
curl --remote-name https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-virt-3.13.5-x86_64.iso
```

Then we can see that the files we are after are in the boot directory
```
% tar -tvf alpine-virt-3.13.5-x86_64.iso 'boot'
dr-xr-xr-x  5 0      0        2048 14 Apr 12:31 boot
dr-xr-xr-x  2 0      0        2048 14 Apr 12:31 boot/dtbs-virt
dr-xr-xr-x  2 0      0        2048 14 Apr 12:31 boot/grub
dr-xr-xr-x  2 0      0        2048 14 Apr 12:31 boot/syslinux
-r--r--r--  1 0      0        2048 14 Apr 12:31 boot/syslinux/boot.cat
-r--r--r--  1 0      0     1474560 14 Apr 12:31 boot/grub/efi.img
-r--r--r--  1 0      0       43008 25 Nov 11:56 boot/syslinux/isolinux.bin
-r--r--r--  1 0      0         176 14 Apr 12:31 boot/grub/grub.cfg
-r--r--r--  1 0      0      123666 14 Apr 12:31 boot/config-virt
-r--r--r--  1 0      0    13193216 14 Apr 12:31 boot/modloop-virt
-r--r--r--  1 0      0         432 25 Nov 11:56 boot/syslinux/isohdpfx.bin
-r--r--r--  1 0      0     6005072 14 Apr 12:31 boot/initramfs-virt
-r--r--r--  1 0      0      116060 25 Nov 11:56 boot/syslinux/ldlinux.c32
-r--r--r--  1 0      0      180148 25 Nov 11:56 boot/syslinux/libcom32.c32
-r--r--r--  1 0      0       23524 25 Nov 11:56 boot/syslinux/libutil.c32
-r--r--r--  1 0      0       11712 25 Nov 11:56 boot/syslinux/mboot.c32
-r--r--r--  1 0      0         248 14 Apr 12:31 boot/syslinux/syslinux.cfg
-r--r--r--  1 0      0     3630128 14 Apr 12:31 boot/System.map-virt
-r--r--r--  1 0      0     6165088 14 Apr 12:31 boot/vmlinuz-virt
```

Extract the files we are after
```
% tar -tvf alpine-virt-*.iso boot/vmlinuz-virt boot/initramfs-virt
x boot/initramfs-virt
x boot/vmlinuz-virt
```

It's useful to understand a bit how the iso would work
if we were able to boot from it directly.
The bootloader, grub, uses syslinux to start the kernel.
We can get some useful insights into what kernel options to pass
by looking at `boot/syslinux/syslinux.cfg`

```
% tar -xvf alpine-virt-*.iso --to-stdout boot/syslinux/syslinux.cfg 2>/dev/null
SERIAL 0 115200
TIMEOUT 20
PROMPT 1
DEFAULT virt

LABEL virt
MENU LABEL Linux virt
KERNEL /boot/vmlinuz-virt
INITRD /boot/initramfs-virt
FDTDIR /boot/dtbs-virt
APPEND modules=loop,squashfs,sd-mod,usb-storage quiet console=tty0 console=ttyS0,115200
```

It will make sense to include that
`modules=loop,squashfs,sd-mod,usb-storage`
in our arguments to hyperkit.

Create a disk file we can install to
```
mkfile 1g alpine.img
```

The easiest way to start up our VM is to use a wrapper script.

```bash
#!/bin/bash

HYPERKIT="hyperkit/build/hyperkit"

# Use own vpnkit process
VPNKIT_SOCK=vpnkit.eth.sock
PIDFILE=vpnkit.pid
vpnkit --ethernet=$VPNKIT_SOCK --log-destination=asl &
echo "$!" > $PIDFILE
trap 'test -f $PIDFILE && kill `cat $PIDFILE` && rm $PIDFILE' EXIT

# Specify a UUID to receive the same network mac address on restarts
UUID="-U deaddead-dead-dead-dead-deaddeadaaaa"

# Linux
# Theses are extracted out of the iso
KERNEL="boot/vmlinuz-virt"
INITRD="boot/initramfs-virt"
CMDLINE="earlyprintk=serial console=ttyS0 modules=loop,squashfs,sd-mod,usb-storage"

# update this for the version you download
BOOTVOLUME="alpine-virt-3.13.5-x86_64.iso"
IMG=alpine.img

# Give the VM 1Gb of RAM
MEM="-m 1G"
# Uncomment to give the VM two CPUs
#SMP="-c 2"
# Using virtio-net (vmnet) requires being root
#NET="-s 2:0,virtio-net"
NET="-s 2:0,virtio-vpnkit,path=$VPNKIT_SOCK"
IMG_CD="-s 3:0,ahci-cd,$BOOTVOLUME"
IMG_HDD="-s 4:0,virtio-blk,$IMG"
PCI_DEV="-s 0:0,hostbridge -s 31,lpc"
# Connect the serial port to stdio
LPC_DEV="-l com1,stdio"
ACPI="-A -u"

"$HYPERKIT" $ACPI $MEM $SMP $PCI_DEV $LPC_DEV $NET $IMG_CD $IMG_HDD $UUID -f kexec,$KERNEL,$INITRD,"$CMDLINE"
```

Running that script should boot you into the Alpine Linux live CD.
Log in as `root` with a blank password and have a little play.

Next time we'll look at installing the OS onto the disk
and the challenges that can come up there.
