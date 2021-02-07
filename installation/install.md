# Arch Linux Installation Guide (LUKS on LVM) - 2021.01.20

This document is a guide for installing Arch Linux using the live system booted from an installation medium made from an official installation image.

Arch Linux should run on any x86_64-compatible machine with a minimum of 512 MiB RAM, though more memory is needed to boot the live system for installation. A basic installation should take less than 2 GiB of disk space. As the installation process needs to retrieve packages from a remote repository, this guide assumes a working internet connection is available.

This installation will be done on a laptop with two hard drives (SSD, HDD), UEFI boot mode, 8GB of RAM and an Intel CPU. We will add features like tmp and swap partitions that will be wiped and re-encrypted with a mew key after every reboot to avoid data leakages; the root partition will require a password to be decrypted and proceed with the boot process, and the home partition will be decrypted automatically with a key file to avoid typing two different passwords whenever we decide to login. To achieve all this, basically the partition layout will be LUKS on top of LVM.

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

| Name |  Partition | Partition type | Size |
| ----:|-----------:|---------------:|-----:|
| efi  |  /dev/sdb1 |  ESP           | 512MB|
| data |  /dev/sdb2 |  Linux LVM     | 500GB|

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

2. Data

       # fdisk /dev/sdb
       # n
       # 2
       # <Enter>
       # <Enter>
       # t
       # 30
       # p
       # w

This is what the logical volumes will look like:

| Name         | Mount Point|     Logical Volume        |  Size    |
| ------------:|-----------:|--------------------------:|---------:|
| lv-cryptswap | none       | /dev/vg-data/lv-cryptswap |       8GB|
| lv-crypttmp  | /mnt/tmp   | /dev/vg-data/lv-crypttmp  |       1GB|
| lv-cryptroot | /mnt       | /dev/vg-data/lv-crypyroot |     150GB|
| lv-crypthome | /mnt/home  | /dev/vg-data/lv=crypthome |  100%FREE|

We will create the all the logical volumes before encrypting anything. The order of creation here is important since /dev/sdb is the SSD drive and I want the OS related stuff to be there for performance reasons.

    # pvcreate /dev/sdb2
    # vgcreate vg-data /dev/sdb2
    # lvcreate -L 8G vg-data -n lv-cryptswap
    # lvcreate -L 1G vg-data -n lv-crypttmp
    # lvcreate -L 150GB vg-data -n lv-cryptroot
    # lvcreate -l 100%FREE vg-data -n lv-crypthome

After the logical volumes were created we can proceed with the encryption, formatting and mounting:

    # echo "root volume setup"
    # cryptsetup luksFormat /dev/vg-data/lv-cryptroot
    # cryptsetup open /dev/vg-data/lv-cryptroot root
    # mkfs.ext4 /dev/mapper/root
    # mount /dev/mapper/root /mnt

    # echo "boot partition setup"
    # dd if=/dev/zero of=/dev/sdb1 bs=1M status=progress
    # mkfs.fat -F32 /dev/sdb1
    # mkdir /mnt/boot
    # mount /dev/sdb1 /mnt/boot

    # echo "home volume setup"
    # mkdir -p -m 700 /mnt/etc/luks-keys
    # dd if=/dev/random of=/mnt/etc/luks-keys/home bs=1 count=256 status=progress
    # cryptsetup luksFormat -v /dev/vg-data/lv-crypthome /mnt/etc/luks-keys/home
    # cryptsetup -d /mnt/etc/luks-keys/home open /dev/vg-data/lv-crypthome home
    # mkfs.ext4 /dev/mapper/home
    # mkdir /mnt/home
    # mount /dev/mapper/home /mnt/home

## Installation

### Select the appropriate mirror

Sometimes downloads from mirrors are way to slow, that’s because the mirrorlist (located in /etc/pacman.d/mirrorlist) has a huge number of mirrors but not in a good order. The top mirror is chosen automatically and it may not always be a good choice, but there is a solution for that: reflector

Get the good mirror list with reflector and save it to mirrorlist. You can change the country from US to your own country.

    # reflector -c "US" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist

### Install base system

The `base` package does not include all tools from the live installation, so installing other packages may be necessary for a fully functional base system.
    

    # pacman -Syy && pacstrap /mnt base base-devel bash-completion linux linux-headers linux-firmware git vim intel-ucode lvm2 mkinitcpio openssh os-prober  wpa_supplicant grub efibootmgr networkmanager network-manager-applet dialog mtools dosfstools python lua bluez bluez-utils cups xdg-utils xdg-user-dirs alsa-utils pulseaudio pulseaudio-bluetooth pulseaudio-alsa pulseaudio-equalizer pulseaudio-jack reflector sudo xf86-video-intel

### Generate fstab file

This file can be used to define how disk partitions, various other block devices, or remote filesystems should be mounted into the filesystem. Generate an fstab file (use `-U` or `-L` to define by UUID or labels, respectively):

    # genfstab -L /mnt >> /mnt/etc/fstab

Also the following entries need to be added to `/mnt/etc/fstab` for the tmp and swap volumes

    /dev/mapper/tmp         /tmp    tmpfs           defaults        0       0
    /dev/mapper/swap        none    swap            sw              0       0

### Update crypttab file

This file `/mnt/etc/crypttab` will be used to handle all the automatic encryption steps for tmp, swap and home volumes

    swap	/dev/vg-data/lv-cryptswap	/dev/urandom	swap,cipher=aes-cbc-essiv:sha256,size=256
    tmp	    /dev/vg-data/lv-crypttmp	/dev/urandom	tmp,cipher=aes-cbc-essiv:sha256,size=256
    home	/dev/vg-data/lv-crypthome   /etc/luks-keys/home

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

    # useradd -m -g users -G wheel -s /bin/bash myusername
    # passwd myusername
    # visudo
    => uncomment %wheel ALL=(ALL) ALL

### Set root password

    # passwd

### Setup initramfs

Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap. For LVM and system encryption we need to modify the `/etc/mkinitcpio.conf` file and recreate the initramfs image for every kernel installed on the system

> **NOTE:** Order is important when setting the HOOKS array.

Now you need to go to the `/etc/mkinitcpio.conf` file, find the uncommented `HOOKS` array and add `lvm2` and `encrypt` in between `block` and `filesystems`

We previously installed only the `linux` kernel, so we need to run this to generate the initramfs image for that kernel:

    # mkinitcpio -p linux

### Prepare bootloader

Choose and install a Linux-capable boot loader. If you have an Intel or AMD CPU, enable `microcode` updates in addition.

    # grub-install --target=x86_64-efi --bootloader-id=GRUB --efi-directory=/boot

In the file `/etc/default/grub` edit the line `GRUB_CMDLINE_LINUX` to:

    cryptdevice=UUID=MyDeviceUUID:root root=/dev/mapper/root

Substitute `MyDeviceUUID` with UUID of the `lv-cryptroot` device. It can be found with:

    # blkid /dev/vg-data/lv-cryptroot

Also uncomment the line:

    # GRUB_ENABLE_CRYPTODISK=y

Generate GRUB's configuration file:

    # grub-mkconfig -o /boot/grub/grub.cfg
    
Enable Network Manager

    # systemctl enable NetworkManager
    # systemctl enable bluetooth
    # systemctl enable cups

Now you are ready to reboot and test your installation:

    # exit
    # umount /mnt
    # reboot
    
After rebooting use nmtui to get connected to the WiFi. 

## Post-installation

1. [Networking](https://github.com/ariguillegp/archlinux/blob/main/networking/wifi.md)
3. TBD ...

## Based on

* [Arch Linux Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide)
* [Arch Linux LUKS on LVM](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LUKS_on_LVM)
* [How to Install Arch Linux](https://itsfoss.com/install-arch-linux/)
* [mjnaderi](https://gist.github.com/mjnaderi/28264ce68f87f52f2cabb823a503e673)
* [huntrar](https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268)
