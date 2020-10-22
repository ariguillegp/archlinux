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

Data disk

| Home /dev/sda1                                      |
| --------------------------------------------------: |
| 500GB                                               |

Root disk

|Root /dev/sdb2                      | Swap /dev/sdb2 | Efi /dev/sdb1 |
|----------------------------------: | --------------:| -------------:|
|                               120GB| 8GB            |          512MB|

For creating the partitions:

1. EFI partition

    # fdisk /dev/sdb
    # n
    # 1
    # <Enter>
    # +512M
    # t
    # 1

2. Root and Swap

    # fdisk /dev/sdb
    # n
    # 2
    # <Enter>
    # <Enter>
    # t
    # 30

3. Home

    # fdisk /dev/sda
    # n
    # 1
    # <Enter>
    # <Enter>
    # t
    # 30

## Installation

## Post-installation
