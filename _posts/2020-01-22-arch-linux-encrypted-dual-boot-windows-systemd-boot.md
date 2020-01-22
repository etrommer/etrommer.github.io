---
layout: post
title: Encrypted Arch/Windows Dual Boot Install
categories:
  - Projects
---

For my new laptop, I wanted a dual-boot solution with Windows 10 and Arch Linux. I carry my laptop around a lot and mainly work on Linux, so I also wanted especially the Linux partition to be fully encrypted. Most of what I used to set up the system was taken from [this Github Gist](https://gist.github.com/mattiaslundberg/8620837) and the official [Arch Linux Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS), however I prefer systemd-boot over GRUB. I also prefer to have everything in one partition. The instructions on how to set things up for my particular use case are spread over a few different places, and there were a few things that took some initial head scratching, which is why I chose to collect everything here.

### Windows Install

From a blank system, install Windows first. During install, partition the disk so that enough space for your Linux system remains. On of the System partition, Windows will create additional partitions during the installation.

### Prepare Arch Linux Live USB

Copy a local image of Arch onto a USB drive. Use `lsblk` to find out, which `/dev/sdX` is your USB drive.

```
dd if=archlinux.img of=/dev/sdX bs=16M && sync
```

### Partition the disk

*Note: From here on, I'll assume that your main drive is under `/dev/sda` and that you use EFI and the same partition scheme as I do. I'll use `vim` for text editing, but you could also use `nano` if you're not as familiar with working in a terminal.*

After booting from the USB drive, run:

```bash
cgdisk /dev/sda
```

to partition the disk. Here's what my partition table looks after the Windows Install:

```
/dev/sda1		529M			Windows recovery environment
/dev/sda2		100M			EFI System
/dev/sda3		16M			Microsoft reserved
/dev/sda4		152.6G			Microsoft basic data
Free Space		312.5G
```

create a new partition of type `8300` in the free space. No other partitions are needed, as I'll be using a [Swap File](https://wiki.archlinux.org/index.php/Swap#Swap_file) instead of a dedicated swap partition. We also don't need a dedicated boot partition, as our Kernel and `systemd-boot` will fit inside the 100M of the EFI partition.

### Encrypt the Linux System Partition

Create an encrypted partition with:

```
cryptsetup -c aes-xts-plain64 -y --use-random luksFormat /dev/sda5
```

type the password that you want your partition encrypted under and confirm. After that, open the partition with

```
# cryptsetup open /dev/sda5 cryptdisk
```

### Set up LVM and a File System

Although not strictly necessary, we'll also create an LVM volume inside our encrypted partition.

```
pvcreate /dev/mapper/cryptdisk
vgcreate arch /dev/mapper/cryptdisk
lvcreate -l +100%FREE arch --name root
```

after that, we can create a file system in our LVM volume

```
mkfs.ext4 /dev/mapper/arch-root
```

and mount our new system disk

```
mkdir /mnt
mount /dev/mapper/arch-root /mnt
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
```

note that we have **not** created a new EFI partition, but rather mounted the one that was created during the Windows install. This is crucial to be able to dual-boot into Windows. Unfortunately, Windows only allocates 100Mb for the EFI, which is not a lot and just barely fits the Linux Kernel and systemd-boot. If you need more space, you'd have to create a larger EFI partition and copy over the data that Windows put there, then delete the old one.

### Basic System Install

Install the basic system packages. This assumes that you are connected to the Internet via Ethernet, which seem the least troublesome option by far.

```
pacstrap /mnt base base-devel linux linux-firmware zsh vim git lvm2
```

after that, generate the [fstab](https://wiki.archlinux.org/index.php/fstab) of the new system

```
genfstab -pU /mnt >> /mnt/etc/fstab
```

and change into the new systems shell

```
arch-chroot /mnt
```

set the time zone, hostname and locale:

```
ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc --utc

echo <Your Host Name> /etc/hostname

echo LANG=en_US.UTF-8 >> /etc/locale.conf
echo LANGUAGE=en_US >> /etc/locale.conf
echo LC_ALL=C >> /etc/locale.conf
```

set a password for root and create a user that can use `sudo`

```
passwd
useradd -m -g users -G wheel -s /bin/zsh USERNAME
passwd USERNAME
```

don't forget to enable members of the group `wheel` to use sudo with

```
EDITOR=vim visudo
```

and uncommenting the line

```
%wheel ALL=(ALL) ALL
```

Also now is a good time to install your preferred [network manager](https://wiki.archlinux.org/index.php/Network_configuration#Network_managers) and enable it via systemd.

### Change kernel modules

We need to add a few things to our initial kernel before we can boot our encrypted disk. Open `/etc/mkinitcpio.conf`.

- add `ext4` to `MODULES`

- add `encrypt`and `lvm2` to `HOOKS` before `filesystems`. 

To generate a new kernel, run:

  ```
  mkinitcpio -p linux  
  ```

### Install systemd-boot

systemd-boot comes with systemd, which is part of the basic Arch install, so no additional packages are required. Install it to your EFI partition with

```
bootctl --path=/boot install
```

systemd-boot will pick up the Windows install automatically and create the according Boot Menu Entry. For our Arch install, we'll still need to create two files.

First, we need a basic loader config under `/boot/loader/loader.conf` with these contents:

```
default arch
timeout 3
editor 0
```

Secondly, we'll need a dedicated boot entry for our Arch Install under `/boot/loader/entries/arch.conf`:

```
title	Arch Linux Encrypted
linux	/vmlinuz-linux
initrd	/initramfs-linux.img
options cryptdevice=UUID=<Your /dev/sda5 UUID>:cryptdisk root=/dev/mapper/arch-root quiet rw
```

you can find the UUID of your `/dev/sda5` partition with `blkid /dev/sda5`. When using Vim, you can copy the output directly into the current buffer with the command `:read ! blkid /dev/sda5`.

### Reboot

Exit the system shell and go back to the live system

```
exit
```

Unmount the partitions

```
umount -R /mnt
```

and reboot

```
reboot
```

That's it. You should now be greeted by two entries that allow you to select whether you want to boot into Windows or Linux. When selecting Linux, you should be prompted for a password that decrypts the disk. Enjoy your new Linux/Windows dual boot system!

### Bonus: Fix system clock
Windows sets the system clock to local time, while Linux prefers to set it to UTC. This means that the displayed system time will be incorrect as soon as you switch between the two. I find that it's easiest to fix this by making [Linux use local time](https://www.howtogeek.com/323390/how-to-fix-windows-and-linux-showing-different-times-when-dual-booting/) as well:

```
timedatectl set-local-rtc 1 --adjust-system-clock
```
