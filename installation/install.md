# Arch Linux Installation Guide - 2020.10.01

This document is a guide for installing Arch Linux using the live system booted from an installation medium made from an official installation image.

Arch Linux should run on any x86_64-compatible machine with a minimum of 512 MiB RAM, though more memory is needed to boot the live system for installation. A basic installation should take less than 2 GiB of disk space. As the installation process needs to retrieve packages from a remote repository, this guide assumes a working internet connection is available.

## Pre-installation

### Get an installation image

We will be installing the OS from an ISO image written to a USB flash drive. In order to get the ISO file visit [Downloads](https://www.archlinux.org/download/) and don't forget to get the checksums file too.

### Verify SHA1 checksum

It is highly recomended to verify the image's checksum before use. We don't wanna end up with a malicious image. Let's assume that `sha1sums.txt` is the file with the sha1 checksums and `archlinux-2020.10.01-x86_64.iso` is our ISO image. The verification can be done like this:

    # cat sha1sums.txt | egrep "$(sha1sum archlinux-2020.10.01-x86_64.iso)"

If there is no output, that's a sign the image was somehow altered and it should not be used.

### Prepare installation medium

Assuming we have the flash drive on `/dev/sdc1` and the Arch Linux ISO under `~/Downloads/`, we can copy all the data to the flash drive using the following command:

    # dd bs=4M if=~/Downloads/archlinux-2020.10.01-x86_64.iso of=/dev/sdc status=progress && sync

After that's done, we are ready to plug the flash drive in the target machine, reboot and start the installation process.

### Boot the live environment

> **NOTE:** Arch Linux installation images do not support **Secure Boot**, so you need to disable it before booting from the flash drive. It can be enabled afterwards.

You will be logged in on the first virtual console as the **root** user and presented with a **zsh** shell prompt.

### Set the keyboard layout

The default console keymap is US, so there is no comments for now here since we will be using that one.

### Verify boot mode

We can quickly check if the system is booted in UEFI or BIOS mode. If the system did not boot in the mode you desired, refer to your motherboard's manual.

    # ls /sys/firmware/efi/efivars

If the command shows the directory without error, then the system is booted in UEFI mode.

### Internet connectivity

For some basic network troubleshooting, follow these items and ensure that you meet them all:

1. Your network interface is listed and enabled. Otherwise check the device driver.

       # ip link

2. You are connected to the network (we will check the wireless scenario in this tutorial -- wlan0 will be out network interface)

       # iwctl

      All the following commands run inside the `iwctl` console and XXXXXXXX will be the SSID of the WiFi network we wanna connect to.

       # [iwctl]: station wlan0 show
       # [iwctl]: station wlan0 connect XXXXXXXX
       # Passphrase: _
       # Ctrl + C

3. Your network interface has an IP address (we will consider DHCP for now).

       # ip a

4. Your routing table is correctly setup.

       # ip r

5. You can ping an IP on your same subnet (e.g. your default gateway).

       # ping <IP of default gateway>

6. You can ping an external IP (e.g. `8.8.8.8` is a convenient one).

       # ping 8.8.8.8

7. You resolve domain names (e.g. `archlinux.org`)

       # ping archlinux.org

### Update the system clock

Let's use NTP for better accuracy

    # timedatectl set-ntp true

### Partition the disks

When recognized by the live system, disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use `lsblk` or `fdisk`.

    # fdisk -l

Results ending in `rom`, `loop` or `airoot` may be ignored.

Now assuming you have a backup of all your files, we are going to delete all the partitions on the machine's hard drives. Our system has 2 hard drives: one will be used for the root partition and the other one for regular user data. Both drives will be encrypted with LVM on LUKS.

To delete partition `/dev/sda1` for example:

    # fdisk /dev/sda
    # m
    # d
    # 1
    # w

The same procedure can be followed to delete any other partition. Now the following table shows the partition layout:

| Name | Mount Point| Partition | Partition type | Size |
| ----:|-----------:|----------:|---------------:|-----:|
| efi  | /mnt/boot/ | /dev/sdb1 |  ESP           | 512MB|
| root | /mnt       | /dev/sdb2 |  Linux LVM     | 128GB|
| home | /mnt/home  | /dev/sda1 |  Linux LVM     | 500GB|


For creating the partitions:

1. EFI partition

       # fdisk /dev/sdb
       # n
       # 1
       # <Enter>
       # +512M
       # t
       # 1
       # p

2. Root and Swap

       # fdisk /dev/sdb
       # n
       # 2
       # <Enter>
       # <Enter>
       # t
       # 30
       # p
       # w

3. Home

       # fdisk /dev/sda
       # n
       # 1
       # <Enter>
       # <Enter>
       # t
       # 30
       # p
       # w

Now we will encrypt the root and home partitions before creating any of the logical volumes that will go over there.

    # cryptsetup luksFormat /dev/sdb2
    # cryptsetup open /dev/sdb2 cryptlvmroot
    # cryptsetup luksFormat /dev/sda1
    # cryptsetup open /dev/sda1 cryptlvmhome

After both devices were encrypted and opened we can proceed on creating the lvm structures:

    # pvcreate /dev/mapper/cryptlvmroot
    # vgcreate vg-root /dev/mapper/cryptlvmroot
    # lvcreate -L 4G vg-root -n lv-swap
    # lvcreate -l 100%FREE vg-root -n lv-root

    # pvcreate /dev/mapper/cryptlvmhome
    # vgcreate vg-home /dev/mapper/cryptlvmhome
    # lvcreate -l 100%FREE vg-home -n lv-home

Next we will format and mount all the volumes we've created:

    # mkfs.ext4 /dev/vg-root/lv-root
    # mkfs.ext4 /dev/vg-home/lv-home
    # mkswap /dev/vg-root/lv-swap

    # mount /dev/vg-root/lv-root /mnt
    # mkdir /mnt/home; mount /dev/vg-home/lv-home !$
    # swapon /dev/vg-root/lv-swap

As out last step in this section we will prepare the boot(EFI) partition

    # mkfs.fat -F32 /dev/sdb1
    # mkdir /mnt/boot; mount /dev/sdb1 !$

## Installation

### Select the appropriate mirror

Sometimes downloads from mirrors are way to slow, that’s because the mirrorlist (located in /etc/pacman.d/mirrorlist) has a huge number of mirrors but not in a good order. The top mirror is chosen automatically and it may not always be a good choice.

Thankfully, there is a fix for that. First sync the pacman repository so that you can download and install software:

    # pacman -Syy

Now, install reflector too that you can use to list the fresh and fast mirrors located in your country:

    # pacman -S reflector

Create a backup of the mirrors list, just in case anything goes sideways

    # cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak

Now, get the good mirror list with reflector and save it to mirrorlist. You can change the country from US to your own country.

    # reflector -c "US" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist

### Install base system

The `base` package does not include all tools from the live installation, so installing other packages may be necessary for a fully functional base system.

    # pacstrap /mnt base linux linux-headers linux-firmware linux-lts linux-lts-headers base-devel vim openssh git mkinitcpio lvm2 os-prober

### Generate fstab file

This file can be used to define how disk partitions, various other block devices, or remote filesystems should be mounted into the filesystem. Generate an fstab file (use `-U` or `-L` to define by UUID or labels, respectively):

    # genfstab -U /mnt >> /mnt/etc/fstab

### Using the installed system

    # arch-chroot /mnt

### Setup Localization

This is what sets the language, numbering, date and currency formats for your system.

Open the file `/etc/locale.gen` using your preferred editor and uncomment (remove the # from the start of the line) the language you prefer. I have used en_US.UTF-8.

Now generate the locale config in `/etc` directory file using the below commands one by one:

    # locale-gen
    # echo LANG=en_US.UTF-8 > /etc/locale.conf
    # export LANG=en_US.UTF-8

Both locale and timezone settings can be changed later on as well when you are using your Arch Linux system.

### Network configuration

Create the `/etc/hostname` file:

    # echo "myhostname" > /etc/hostname

Add matching entries to `/etc/hosts` file:

    127.0.0.1 localhost
    ::1 localhost
    127.0.1.1 myhostname.localdomain myhostname

### Create regular user with sudo privileges

    # pacman -S sudo
    # useradd -m -g users -G wheel -s /bin/bash myusername
    # passwd myusername
    # visudo
    => uncomment %wheel ALL=(ALL) ALL

### Set root password

    # passwd

### Setup initramfs

Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap. For LVM and system encryption we need to modify the `/etc/mkinitcpio.conf` file and recreate the initramfs image for every kernel installed on the system

> **NOTE:** Order is important when setting the HOOKS array.

Now you need to go to the `/etc/mkinitcpio.conf` file, find the uncommented `HOOKS` array and add `encrypt` and `lvm2` in between `block` and `filesystems`

We previously installed `linux` and `linux-lts` kernels, so we need to run this:

    # mkinitcpio -p linux
    # mkinitcpio -p linux-lts

### Prepare bootloader

Choose and install a Linux-capable boot loader. If you have an Intel or AMD CPU, enable `microcode` updates in addition.

    # pacman -S grub efibootmgr
    # grub-install --target=x86_64-efi --bootloader-id=GRUB --efi-directory=/boot

In the file `/etc/default/grub` edit the line `GRUB_CMDLINE_LINUX` to:

    cryptdevice=UUID=device-UUID:cryptlvmroot root=/dev/vg-root/lv-root

Substitute `device-UUID` with UUID of the `lv-root` device. You can find it on `etc/fstab`

If you have an Intel or AMD CPU, enable microcode updates in addition.

    # pacman -S intel-ucode (only for Intel CPUs)
    # pacman -S amd-ucode (only for AMD CPUs)

Generate GRUB's configuration file:

    # grub-mkconfig -o /boot/grub/grub.cfg

### Install desktop environment

Install the windows manager first:

    # pacman -S xorg

Next you can install the desktop environment (Gnome) and enable the services if you want

    # pacman -S gnome
    # systemctl start gdm.service
    # systemctl enable gdm.service

Network manager is also very useful, so we will enable it as well

    # systemctl enable NetworkManager.service

Now you are ready to reboot and test your installation:

    # reboot

## Post-installation
