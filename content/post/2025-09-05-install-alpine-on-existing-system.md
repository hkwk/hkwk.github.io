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
sda     253:0    0   30G  0 disk
├─sda1  253:1    0  106M 0 part /boot/efi
└─sda2  253:2   0  29.9G  0 part /

root@hkt:~# cat /etc/netplan/50-network.yaml
# network-config
network:
    version: 2
    ethernets:
        ens17:
            addresses: ['ip.add.re.ss/24']
            gateway4: ga.te.w.ay
            nameservers:
                addresses: ['8.8.8.8','1.1.1.1']
```

## Mount root

```bash
mkdir -p /tmp/alpine
cd /tmp/alpine
wget https://raw.githubusercontent.com/cemkeylan/genfstab/master/genfstab
wget https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-minirootfs-3.22.1-x86_64.tar.gz
tar xpf alpine-minirootfs-3.22.1-x86_64.tar.gz --xattrs-include='*.*' --numeric-owner


```

## Mount the needed fake filesystems

```bash
mount --types proc /proc /tmp/alpine/proc
mount --rbind /sys /tmp/alpine/sys
mount --make-rslave /tmp/alpine/sys
mount --rbind /dev /tmp/alpine/dev
mount --make-rslave /tmp/alpine/dev
mount --bind /run /tmp/alpine/run
mount --make-slave /tmp/alpine/run
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm /run/shm

```

##  1st Chroot and Setup Virtual System

```bash
chroot /tmp/alpine/ /bin/ash

export PS1='root@alpine-chroot-1 # '

tee /etc/resolv.conf <<EOF
nameserver 1.1.1.1
EOF

apk update
apk add tar mount

mkdir /mnt/alpine
mount /dev/sda2 /mnt/alpine
mkdir /mnt/alpine/boot/efi
mount /dev/sda1 /mnt/alpine/boot/efi

cp /genfstab /mnt/alpine
cp /alpine-minirootfs-3.22.1-x86_64.tar.gz /mnt/alpine

cd /mnt/alpine
rm -rf bin home etc lib  lib64 opt root sbin srv usr var
tar xpvf alpine-minirootfs-3.22.1-x86_64.tar.gz --xattrs-include='*.*' --numeric-owner

mount --types proc /proc /mnt/alpine/proc
mount --rbind /sys /mnt/alpine/sys
mount --make-rslave /mnt/alpine/sys
mount --rbind /dev /mnt/alpine/dev
mount --make-rslave /mnt/alpine/dev
mount --bind /run /mnt/alpine/run
mount --make-slave /mnt/alpine/run
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm /run/shm
```

##  2nd Chroot and Setup Real Alpine System

```bash
chroot /mnt/alpine/ /bin/ash

export PS1='root@alpine-chroot-2 # '
export PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
mkdir /lib/modules

tee /etc/resolv.conf <<EOF
nameserver 1.1.1.1
EOF

apk update
apk add alpine-conf openrc e2fsprogs --no-cache

setup-alpine

# Which NTP client to run? ('busybox', 'openntpd', 'chrony' or 'none') [busybox] chrony
# Allow root ssh login? ('?' for help) [prohibit-password] yes
# Which disk(s) would you like to use? (or '?' for help or 'none') [none]

apk add grub grub-efi efibootmgr linux-lts

sh /genfstab -U / >> /etc/fstab

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ALPINE
grub-mkconfig -o /boot/grub/grub.cfg

echo 'GRUB_CMDLINE_LINUX="console=ttyS0,19200n8 net.ifnames=0 modules=sd-mod,usb-storage,ext4 quiet rootfstype=ext4"' >> /etc/default/grub
update-grub

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
rc-update add sshd boot
```


#### Enjoying Alpine!

refer:

1.[Alpine Semi-Automatic Installation](https://docs.alpinelinux.org/user-handbook/0.1a/Installing/manual.html)

2.[How to manually install alpine linux on any linux distribution](https://blog.ari.lt/b/how-to-manually-install-alpine-linux-on-any-linux-distribution/)

3.[Replacing non-Alpine Linux with Alpine remotely](https://wiki.alpinelinux.org/wiki/Replacing_non-Alpine_Linux_with_Alpine_remotely#Without_VNC_access)

4.[ Booting issues after upgrade (mounting /dev/sda on /sysroot failed)](https://www.reddit.com/r/AlpineLinux/comments/m99ksm/booting_issues_after_upgrade_mounting_devsda_on/)