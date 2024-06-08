---
layout: post
title: Running Linux in QEMU - s/hyperkit/qemu/g
date: 2024-06-08
---

Ok, it's three years since I wrote about running [Linux in Hyperkit][1],
we've all traded our amd64 macbooks in for the cooler arm64 version
and we've noticed Hyperkit is dead.
How're we gonna run our little pet virtual machines now?

<!-- TODO: link to previous post -->
[1]: /2021/05/02/running-linux-in-hyperkit.html

Is it a bird, is it a... oh wait, yes, it's a bird!
[QEMU][2] is able to save us.

[2]: https://www.qemu.org/

## Getting QEMU

QEMU can [be got][3] with [Homebrew][4]:

[3]: https://www.qemu.org/download/#macos
[4]: https://brew.sh/

```shell
brew install qemu
```

## Running a Linux

QEMU has some special options for booting a Linux VM,
the `-kernel` option along with `-append` and `-initrd`.
i.e.

```
qemu-system-aarch64 ... -kernel vmlinuz-virt -initrd initramfs-virt -append 'quiet console=tty0'
```

Erm... What's a vmlinuz and where do we get one?
Great question!
It's a compressed Linux kernel image and we get it from the CD.
Where have you been for the ...
Oh, you don't have a CD, or a CD drive?
Worry not! It's just a conceptual CD.

Let's use little Alpine Linux for our example.
Download the aarch64 Virtual ISO from <https://alpinelinux.org/downloads/>
e.g.

```
curl --remote-name https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-virt-3.20.0-aarch64.iso
```

Then we can see the files we are after are in the boot directory

```
% tar -tvf alpine-virt-3.20.0-aarch64.iso boot | grep -v 'dtbs-virt'
dr-xr-xr-x  4 0      0        2048 22 May 11:52 boot
dr-xr-xr-x  2 0      0        2048 22 May 11:52 boot/grub
-r--r--r--  1 0      0     1474560  7 May 22:04 boot/grub/efi.img
-r--r--r--  1 0      0      148997 22 May 11:52 boot/config-6.6.31-0-virt
-r--r--r--  1 0      0         171 22 May 11:52 boot/grub/grub.cfg
-r--r--r--  1 0      0    15728640 22 May 11:52 boot/modloop-virt
-r--r--r--  1 0      0     9517748 22 May 11:52 boot/initramfs-virt
-r--r--r--  1 0      0     3126395 22 May 11:52 boot/System.map-6.6.31-0-virt
-r-xr-xr-x  1 0      0     9150976 22 May 11:52 boot/vmlinuz-virt
```

Extract the files we are after

```
% tar -xvf alpine-virt-*.iso boot/vmlinuz-virt boot/initramfs-virt
x boot/initramfs-virt
x boot/vmlinuz-virt
```

Now what about that `-append`, have we forgotten about that?
No, no, ... think ... how would the CD work if we had a CD drive?
GRUB of course!
Let's look at the `grub.cfg`

```
% tar -xf alpine-virt-*.iso --to-stdout boot/grub/grub.cfg
set timeout=1

menuentry "Linux virt" {
linux	/boot/vmlinuz-virt modules=loop,squashfs,sd-mod,usb-storage quiet console=tty0 console=ttyAMA0
initrd	/boot/initramfs-virt
}
```

Maybe it will be sensible to include that
`modules=loop,squashfs,sd-mod,usb-storage`
in our arguments to QEMU.

Let's now create a disk to install Alpine to

```
qemu-img create -f qcow2 -o compression_type=zstd alpine.qcow2 4G
```

The easiest way to start our VM will be to use a wrapper script

```
% cat run-installer.sh
#!/bin/sh

# Extracted out of the iso
KERNEL="boot/vmlinuz-virt"
INITRD="boot/initramfs-virt"
CMDLINE="modules=loop,squashfs,sd-mod,usb-storage earlyprintk=serial console=tty0 console=ttyAMA0"

# Change to the version you downloaded
BOOTVOLUME=alpine-virt-3.20.0-aarch64.iso
IMG=alpine.qcow2

# Name what you like
NAME="pet-alpine"
# Generate your own with:
#   python3 -c 'import uuid; print(str(uuid.uuid4()).upper())'
UUID=D46345C3-E056-4D97-A752-B129BED48703

exec qemu-system-aarch64 \
    -nodefaults \
    -vga none \
    -device virtio-net-pci,netdev=net0 \
    -netdev user,id=net0 \
    -nographic \
    -chardev stdio,id=term0 \
    -serial chardev:term0 \
    -cpu host \
    -smp cpus=4,sockets=1,cores=4,threads=1 \
    -machine virt,highmem=off \
    -accel hvf \
    -m 2G \
    -device nec-usb-xhci,id=usb-bus \
    -device usb-storage,drive=drive0,removable=true,bootindex=0,bus=usb-bus.0 \
    -drive if=none,media=cdrom,id=drive0,file=$BOOTVOLUME,readonly=on \
    -device virtio-blk-pci,drive=drive1,bootindex=1 \
    -drive if=none,media=disk,id=drive1,file=$IMG \
    -name "$NAME" -uuid $UUID \
    -kernel $KERNEL -append "$CMDLINE" -initrd $INITRD
```

Change the values as needed.

Running that should boot you into the Alpine live CD.
Log in as `root` with a blank password and feel your way around.


At this point you can run `setup-alpine` to install the system
to the disk we created earlier.

Select the `vda` disk to install to and `sys` layout, when it gets to that
point.

After the installer says `Installation is complete. Please reboot.`,
don't reboot!

We're going to need to change some parameters
in order to boot from the qcow2 disk correctly.
We can find the parameters by looking in the boot volume.
This time extlinux seems to be the bootloader.

```
# mkdir /boot
# mount /dev/vda1 /boot
# cat /boot/extlinux/extlinux.conf
menu title Alpine Linux
timeout 50
default virt

label virt
menu label Linux virt
kernel /vmlinuz-virt
initrd /initramfs-virt
fdtdir /dtbs-virt
append root=UUID=bd5dc486-ed0f-4dae-93a7-39884acacfda modules=sd-mod,usb-storage,ext4 quiet rootfstype=ext4
```

That final append line looks handy!

<!-- TODO: add 9p args to the first run so we can copy out the initrd ? -->

Now, instead of rebooting, we want to `poweroff` so that we can change
some arguments.

We can create a `run.sh` script by copying the `run-installer.sh` script,
only changing the `CMDLINE` to include the pieces needed to find the boot
partition

```
CMDLINE="root=UUID=bd5dc486-ed0f-4dae-93a7-39884acacfda modules=sd-mod,usb-storage,ext4 rootfstype=ext4 earlyprintk=serial console=tty0 console=ttyAMA0"
#                  ^----- this may differ for you ----^
```

You'll need to adapt the UUID to what you saw in
`/boot/extlinux/extlinux.conf`.


Now we can boot into the installed system

```
% ./run.sh
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x610f0000]
...

Welcome to Alpine Linux 3.20
Kernel 6.6.31-0-virt on an aarch64 (/dev/ttyAMA0)

localhost login:
```

