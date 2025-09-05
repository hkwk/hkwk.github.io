---
author: HKL
categories:
- Default
date: "2025-09-05T07:42:00Z"
slug: install-alpine-on-existing-system
status: publish
tags:
- Linux
- Operating
title: Install Alpine ON Existing Linux system without VNC
---


NO WARRANTY!!! BACKUP ALL YOUR DATA BEFORE OPERATING!
Operating the following In the Exsiting Linux System

Install Alpine ON Existing Linux system without VNC

## Print the current Disk Partation table

```bash
# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   30G  0 disk
├─vda1  253:1    0 29.9G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  106M  0 part /boot/efi

root@hkt:~# cat /etc/netplan/50-network.yaml
# network-config
network:
    version: 2
    ethernets:
        lo:
            addresses: [ '127.0.0.1/8' ]
        ens17:
            addresses: ['ip.add.re.ss/24']
            gateway4: ga.te.w.ay
            nameservers:
                addresses: ['8.8.8.8','1.1.1.1']
```

## Mount root

```bash
root@hkt:~# mkdir -p /mnt/alpine
root@hkt:~# mount /dev/vda1 /mnt/alpine
root@hkt:~# mkdir -p /mnt/alpine/boot/efi
root@hkt:~# mount /dev/vda15 /mnt/alpine/boot/efi
root@hkt:~# cd /mnt/alpine
root@hkt:/mnt/alpine#
root@hkt:/mnt/alpine# wget https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.1-x86_64.tar.gz

root@hkt:/mnt/alpine# tar xpf alpine-minirootfs-3.22.1-x86_64.tar.gz --xattrs-include='*.*' --numeric-owner

root@hkt:/mnt/alpine# wget https://raw.githubusercontent.com/cemkeylan/genfstab/master/genfstab

root@hkt:/mnt/alpine# sh genfstab -U /mnt/alpine >>/mnt/alpine/etc/fstab
```

## Mount the needed fake filesystems

```bash
mount --types proc /proc /mnt/alpine/proc
mount --rbind /sys /mnt/alpine/sys
mount --make-rslave /mnt/alpine/sys
mount --rbind /dev /mnt/alpine/dev
mount --make-rslave /mnt/alpine/dev
mount --bind /run /mnt/alpine/run
mount --make-slave /mnt/alpine/run


tee /mnt/alpine/etc/resolv.conf <<EOF
nameserver 1.1.1.1
nameserver 8.8.8.8
EOF
```

##  Chroot and Setup Alpine

```bash
chroot /mnt/alpine /bin/ash

export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
apk update
apk add alpine-conf openrc e2fsprogs --no-cache

setup-alpine

apk add grub grub-efi efibootmgr linux-lts	

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ALPINE --no-nvram
grub-mkconfig -o /boot/grub/grub.cfg

rc-update add hostname boot
rc-update add devfs sysinit
rc-update add cgroups sysinit
rc-update add bootmisc boot
rc-update add binfmt boot
rc-update add fsck boot
rc-update add seedrng boot
rc-update add root boot
rc-update add procfs boot
rc-update add swap boot
rc-update add sysfs sysinit
rc-update add localmount boot
rc-update add sysctl boot
rc-update add ssh boot

reboot
```

#### Enjoying Alpine!

refer:

1.[Alpine Semi-Automatic Installation](https://docs.alpinelinux.org/user-handbook/0.1a/Installing/manual.html)

2.[How to manually install alpine linux on any linux distribution](https://blog.ari.lt/b/how-to-manually-install-alpine-linux-on-any-linux-distribution/)

3.[Replacing non-Alpine Linux with Alpine remotely](https://wiki.alpinelinux.org/wiki/Replacing_non-Alpine_Linux_with_Alpine_remotely#Without_VNC_access)
