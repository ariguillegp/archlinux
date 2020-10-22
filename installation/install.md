# Arch Linux Installation Guide - 2020.10.01

This document is a guide for installing Arch Linux using the live system booted from an installation medium made from an official installation image.

Arch Linux should run on any x86_64-compatible machine with a minimum of 512 MiB RAM, though more memory is needed to boot the live system for installation. A basic installation should take less than 2 GiB of disk space. As the installation process needs to retrieve packages from a remote repository, this guide assumes a working internet connection is available.

## Pre-installation

### Get an installation image

We will be installing the OS from an ISO image written to a USB drive. In order to get the ISO file visit [Downloads](https://www.archlinux.org/download/) and don't forget to get the PGP signature file too.

### Verify SHA1 checksum

It is highly recomended to verify the image's checksum before use. We don't wanna end up with a malicious image. Let's assume that sha1sums.txt is the file with the sha1 checksums and archlinux-2020.10.01-x86_64.iso is our ISO image. The verification can be done like this:

    # cat sha1sums.txt | egrep "$(sha1sum archlinux-2020.10.01-x86_64.iso)"

If there is no output, that's a sign the image was somehow altered and it should not be used.

### Prepare installation medium

## Installation

## Post-installation
