# Comparison: This Repo vs Omarchy

## Executive Summary

This document compares the manual ArchLinux installation approach documented in this repository with Omarchy's automated installation system. The two approaches represent fundamentally different philosophies: **manual control vs. opinionated automation**.

---

## Overview

### This Repository (Manual Approach)
- **Type**: Documentation-only reference guide
- **Date**: January 2021
- **Philosophy**: Complete manual control, step-by-step commands
- **Target**: Users who want to understand every step
- **Encryption**: LUKS on LVM
- **Time Investment**: Several hours + ongoing configuration
- **Maintenance**: Manual updates required

### Omarchy (Automated Approach)
- **Type**: Fully automated installation system
- **Maintained By**: Basecamp/37signals (DHH)
- **Philosophy**: Opinionated, batteries-included approach
- **Target**: Developers who want a working system immediately
- **Encryption**: LUKS with BTRFS (compressed)
- **Time Investment**: 5-30 minutes
- **Maintenance**: Migration system handles updates

---

## Detailed Comparison

### 1. Installation Method

#### This Repo
```bash
# Manual commands, one at a time
cryptsetup luksFormat /dev/vg-data/lv-cryptroot
cryptsetup open /dev/vg-data/lv-cryptroot root
mkfs.ext4 /dev/mapper/root
mount /dev/mapper/root /mnt
# ... hundreds more commands
```

**Characteristics:**
- ~100+ manual commands to execute
- Each step requires understanding
- Easy to make mistakes
- Full transparency of every action
- Steep learning curve
- Documentation only (no scripts)

#### Omarchy
```bash
# Single command installation
wget -qO- https://omarchy.org/install | bash
# Or minimal version:
wget -qO- https://omarchy.org/install-bare | bash
```

**Characteristics:**
- One command, 5-30 minute wait
- Automated everything
- Modular architecture (boot.sh + install.sh)
- 5 distinct installation phases
- Error recovery with retry capability
- Migration system for updates
- Pre-configured with sensible defaults

---

### 2. Encryption Strategy

#### This Repo: LUKS on LVM (Advanced Multi-Layer)

**Architecture:**
```
Physical Disk (sdb2)
  └── LVM Physical Volume
      └── Volume Group (vg-data)
          ├── lv-cryptswap  (8GB)   → Ephemeral encryption
          ├── lv-crypttmp   (1GB)   → Ephemeral encryption
          ├── lv-cryptroot  (150GB) → Password-protected
          └── lv-crypthome  (rest)  → Key file decryption
```

**Security Features:**
- **Root**: Password required at boot
- **Home**: Auto-decrypted with key file (`/etc/luks-keys/home`)
- **Swap**: Re-encrypted with random key every boot (prevents hibernation data leakage)
- **Tmp**: Re-encrypted with random key every boot (prevents temp file persistence)
- **File System**: ext4 (traditional, proven)

**Advantages:**
- Maximum security for tmp/swap (ephemeral)
- Single password at boot (home auto-unlocks)
- Prevents data leakage through swap files
- LVM flexibility (easy resizing, snapshots)

**Disadvantages:**
- Complex setup (LVM + LUKS layers)
- More moving parts to maintain
- Swap re-encryption = no hibernation support
- Larger overhead (LVM + LUKS)

#### Omarchy: LUKS + BTRFS (Modern Simplified)

**Architecture:**
```
Physical Disk
  ├── Partition 1: /boot (unencrypted EFI)
  └── Partition 2: LUKS encrypted
      └── BTRFS with subvolumes
          ├── @ (root)
          ├── @home
          ├── @snapshots
          └── Others
```

**Security Features:**
- Single LUKS password at boot
- Auto-login after disk decryption
- BTRFS compression (zstd) enabled
- Subvolume architecture for flexibility

**Advantages:**
- Simpler architecture (fewer layers)
- BTRFS features (snapshots, compression, COW)
- Transparent compression saves disk space
- Modern filesystem with active development
- Easier to understand and maintain

**Disadvantages:**
- No ephemeral swap/tmp (potential data leakage)
- Auto-login reduces post-boot security
- Single point of failure (no LVM layer)
- Less granular encryption control

---

### 3. Partition Layout

#### This Repo
```
/dev/sdb1    512MB   EFI System Partition
/dev/sdb2    500GB   LVM Physical Volume
  ├── lv-cryptswap    8GB
  ├── lv-crypttmp     1GB
  ├── lv-cryptroot  150GB
  └── lv-crypthome  341GB (remaining)
```

**Philosophy**: Maximum security with separate encrypted volumes for different security zones

#### Omarchy
```
/dev/nvme0n1p1    EFI System Partition
/dev/nvme0n1p2    LUKS encrypted BTRFS
  ├── @ (root)
  ├── @home
  ├── @log
  ├── @cache
  └── @snapshots
```

**Philosophy**: Simplicity with BTRFS subvolumes for organization and snapshot capability

---

### 4. Package Installation

#### This Repo: Minimal Base
```bash
pacstrap /mnt base base-devel bash-completion linux \
  linux-headers linux-firmware git vim intel-ucode \
  lvm2 mkinitcpio openssh os-prober wpa_supplicant \
  grub efibootmgr networkmanager network-manager-applet \
  dialog mtools dosfstools python lua bluez bluez-utils \
  cups xdg-utils xdg-user-dirs alsa-utils pulseaudio \
  pulseaudio-bluetooth pulseaudio-alsa pulseaudio-equalizer \
  pulseaudio-jack reflector sudo xf86-video-intel
```

**Included:**
- Bare minimum system packages
- Basic audio (PulseAudio)
- Basic networking
- No desktop environment
- No applications
- No configuration

**Post-Install Required:**
- Install desktop environment manually
- Install applications one by one
- Configure everything from scratch
- Set up dotfiles
- Configure window manager/DE
- Install development tools
- Configure themes/fonts

#### Omarchy: Complete Development Environment

**Automated Installation Includes:**
- **Base System**: Arch + all necessary drivers
- **Desktop**: Hyprland (Wayland compositor) fully configured
- **Development Tools**: Neovim, Git, compilers, etc.
- **Applications**: Spotify, Pinta, LocalSend, OBS Studio
- **Fonts**: Complete font collection
- **Themes**: 11 pre-configured themes
- **Configuration**: Dotfiles, keybindings, everything ready

**Installation Phases:**
1. **Packaging Phase**: 5 specialized scripts install all packages
2. **Configuration Phase**: 15 core + 7 hardware-specific scripts
3. **Migration System**: State tracking for consistency
4. **Hardware Detection**: Automatic hardware-specific setup
5. **Security Configuration**: Automated security hardening

**Result**: Turn on computer → Enter LUKS password → Working dev environment

---

### 5. Configuration Management

#### This Repo
- **Method**: None (manual configuration required)
- **Dotfiles**: User must create/manage themselves
- **Updates**: Manual intervention for every change
- **Consistency**: No guarantees across installations
- **Documentation**: Step-by-step guide only

#### Omarchy
- **Method**: Git-based configuration management
- **Dotfiles**: Pre-configured and version-controlled
- **Updates**: Migration system with state tracking
- **Consistency**: Identical setup across installations
- **Recovery**: `bash ~/.local/share/omarchy/install.sh` to retry
- **Modular**: Separate scripts for different components

---

### 6. Window Manager / Desktop Environment

#### This Repo
- **Included**: None
- **User Choice**: Completely open
- **Configuration**: 100% manual

Popular choices users typically install:
- i3/i3-gaps (tiling)
- GNOME (full DE)
- KDE Plasma (full DE)
- Sway (Wayland tiling)

#### Omarchy
- **Included**: Hyprland (Wayland compositor)
- **Pre-configured**: Complete keybindings, rules, animations
- **Themes**: 11 ready-to-use themes
- **Opinionated**: DHH's preferred setup
- **Modern**: Wayland-native (not X11)

---

### 7. Maintenance & Updates

#### This Repo
```bash
# Manual updates
pacman -Syu

# User responsible for:
- Checking for breaking changes
- Updating configurations
- Handling conflicts
- Testing everything
```

**Characteristics:**
- Pure Arch Linux rolling release
- Manual intervention for issues
- No automated config updates
- User handles all edge cases
- Documentation may become outdated (2021)

#### Omarchy
```bash
# Automated update handling
# Migration system tracks state
~/.local/state/omarchy/migrations/

# Features:
- Automatic config migrations
- State tracking prevents duplicates
- Recovery on failure
- Modular update scripts
```

**Characteristics:**
- Arch Linux rolling release + Omarchy updates
- Migration system handles config changes
- Version upgrade pathways
- Active maintenance (2025)
- Community support via GitHub

---

### 8. Target Audience

#### This Repo: Arch Purists
**Best For:**
- Users who want to learn Linux deeply
- Those who need custom configurations
- Security-conscious users wanting full control
- People building specialized systems
- Educational purposes
- Understanding what's "under the hood"

**Not Ideal For:**
- Users wanting quick setup
- Those unfamiliar with Linux internals
- People who prefer "just works"
- Developers focused on coding, not system config

#### Omarchy: Pragmatic Developers
**Best For:**
- Developers who want productivity immediately
- Users comfortable with opinionated setups
- Those who trust DHH's choices
- MacOS users transitioning to Linux
- People wanting Arch without the setup pain
- Hyprland enthusiasts

**Not Ideal For:**
- Users wanting custom window managers
- Those opposed to automation
- People wanting minimal systems
- Users who distrust downloaded scripts
- Those needing non-standard configurations

---

### 9. Boot Process

#### This Repo
```
Power On
  ↓
GRUB asks for LUKS password (root)
  ↓
System decrypts root partition
  ↓
System auto-decrypts home (key file)
  ↓
System creates ephemeral swap/tmp
  ↓
Login prompt (requires username + password)
  ↓
User logs in manually
```

**Security:** Two-factor (LUKS password + user password)

#### Omarchy
```
Power On
  ↓
System asks for LUKS password
  ↓
System decrypts everything
  ↓
Automatic login to desktop
  ↓
User immediately in Hyprland
```

**Security:** Single-factor (LUKS password only)
**Philosophy**: Disk encryption is sufficient; auto-login for convenience

---

### 10. Time Investment

#### This Repo
**Initial Setup:**
- Reading documentation: 1-2 hours
- Installation: 2-4 hours (for beginners)
- Desktop environment setup: 2-4 hours
- Application installation: 1-2 hours
- Configuration/theming: 4-10 hours
- **Total: 10-20+ hours**

**Ongoing:**
- System updates: 15-30 min/week
- Config management: Ongoing as needed
- Troubleshooting: Variable

#### Omarchy
**Initial Setup:**
- Installation: 5-30 minutes (automated)
- Customization: 0-2 hours (optional)
- **Total: 5-150 minutes**

**Ongoing:**
- System updates: 15-30 min/week
- Migration system handles config updates
- Less troubleshooting (standardized setup)

---

## Philosophical Differences

### This Repo: The Arch Way
> **"Simple is better than complicated"**

- User makes every decision
- Full transparency
- Minimal base system
- User adds only what they need
- Complete understanding required
- "Do it yourself" philosophy

### Omarchy: The Rails Way
> **"Convention over configuration"**

- Sensible defaults chosen for you
- "It just works" out of the box
- Opinionated but changeable
- Productivity over learning
- Trust the maintainer's choices
- "Done is better than perfect"

---

## Security Comparison

### Strengths of This Repo's Approach

1. **Ephemeral Tmp/Swap**:
   - Prevents data leakage through temp files
   - Swap re-encrypted on every boot
   - No hibernation data persistence

2. **Separate Encryption Zones**:
   - Root and home on different LUKS volumes
   - Granular control over each volume
   - Can have different encryption parameters

3. **Manual Verification**:
   - User sees every command
   - Can verify each step
   - No "black box" automation
   - ISO checksum verification documented

### Strengths of Omarchy's Approach

1. **BTRFS Snapshots**:
   - Easy rollback if something breaks
   - Subvolume management
   - System recovery capabilities

2. **Standardized Security**:
   - Security hardening scripts
   - Consistent across installations
   - Community-reviewed automation

3. **Active Maintenance**:
   - Security updates in automation
   - Modern best practices
   - Active community finding issues

### Weaknesses

#### This Repo
- Documentation from 2021 (potentially outdated)
- No automated security updates to the guide
- User errors in manual execution
- Swap/tmp security = no hibernation

#### Omarchy
- Auto-login reduces post-boot security
- Less granular encryption control
- Trust required in automation scripts
- Opinionated choices may not fit all needs

---

## When to Choose Each Approach

### Choose This Repo's Manual Approach When:

1. **Learning is the goal**: You want to deeply understand Linux
2. **Custom requirements**: Your needs don't fit standard patterns
3. **Security paranoia**: You need ephemeral swap/tmp
4. **Audit requirements**: You must verify every command
5. **No network during install**: Pure offline installation possible
6. **Specific partition layout**: Complex multi-disk setups
7. **Distrust automation**: You want to see everything happening

### Choose Omarchy When:

1. **Productivity is priority**: You want to code, not configure
2. **Hyprland appeals to you**: You like the Wayland compositor approach
3. **Trust DHH's opinions**: You're comfortable with opinionated software
4. **Quick setup needed**: New machine needs to be ready fast
5. **Standard hardware**: Common laptop/desktop configurations
6. **Modern filesystem**: BTRFS features are valuable to you
7. **Active community**: You want ongoing updates and improvements

---

## Hybrid Approach Possibilities

You could combine elements of both:

1. **Manual install + Omarchy config**: Use this repo's security approach, then install Omarchy's dotfiles
2. **Omarchy base + custom partitioning**: Use archinstall with manual partitions, then run Omarchy
3. **Learn then automate**: Follow this guide first, then create your own automation
4. **Fork Omarchy**: Modify Omarchy's scripts for your security requirements

