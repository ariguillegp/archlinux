# Hybrid Approach: Omarchy + Advanced Security

## Overview

This guide combines:
- ✅ **Your security approach**: LUKS on LVM with ephemeral swap/tmp
- ✅ **Omarchy's productivity**: Hyprland, apps, configs, themes

## Strategy

**Install in two phases:**
1. **Phase 1**: Manual base installation with your security preferences
2. **Phase 2**: Install Omarchy's configuration layer on top

---

## Why This Approach Works

Omarchy is fundamentally:
- A collection of configuration scripts
- Pre-installed applications
- Dotfiles and themes
- Hyprland setup

It **doesn't require** its specific partition layout to function. We can install it on any working Arch system.

---

## Prerequisites

1. Read the [manual installation guide](install.md)
2. Understand LUKS on LVM setup
3. Comfortable with command line
4. Have backup of any important data

---

## Phase 1: Secure Base Installation

### Step 1: Follow Your Manual Guide Through Base Install

Follow your existing [install.md](install.md) from start through "Install base system", but with **one key modification**:

#### Modified Package List

Instead of your minimal package list, install a broader base that includes what Omarchy expects:

```bash
# Enhanced base system for Omarchy compatibility
pacstrap /mnt \
    base base-devel linux linux-firmware linux-headers \
    git vim neovim \
    intel-ucode \
    lvm2 \
    grub efibootmgr \
    networkmanager iwd \
    pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
    bluez bluez-utils \
    cups \
    reflector sudo \
    mesa vulkan-intel \
    xdg-utils xdg-user-dirs \
    man-db man-pages texinfo \
    bash-completion \
    openssh \
    wget curl \
    unzip zip \
    htop \
    mkinitcpio
```

**Why these packages?**
- Base system that Omarchy expects
- Audio: Pipewire (modern, not PulseAudio)
- Graphics: Mesa/Vulkan for modern compositors
- Network: NetworkManager + iwd
- Utilities Omarchy scripts may need

### Step 2: Complete Your Encryption Setup

Follow your guide **exactly** for:
- ✅ LUKS on LVM partition layout
- ✅ Ephemeral swap configuration
- ✅ Ephemeral tmp configuration
- ✅ Home partition with key file
- ✅ Crypttab setup
- ✅ Fstab configuration

**Your partition layout remains:**
```
/dev/sdb1         512MB    EFI
/dev/sdb2         500GB    LVM
  ├── lv-cryptswap    8GB    (ephemeral)
  ├── lv-crypttmp     1GB    (ephemeral)
  ├── lv-cryptroot  150GB    (password LUKS)
  └── lv-crypthome   rest    (key file LUKS)
```

### Step 3: Complete Base Configuration

Continue with your guide:
- ✅ Generate fstab
- ✅ Update crypttab
- ✅ Chroot into system
- ✅ Setup localization
- ✅ Network configuration
- ✅ Create user with sudo
- ✅ Setup initramfs (with encrypt + lvm2 hooks)
- ✅ Install and configure GRUB
- ✅ Enable services

### Step 4: First Boot & Verification

```bash
# Reboot into new system
exit
umount -R /mnt
reboot

# After booting, verify encryption
lsblk -f
swapon --show
mount | grep tmp

# Verify network
ping archlinux.org

# Update system
sudo pacman -Syu
```

**CRITICAL**: Don't proceed to Phase 2 until this works perfectly.

---

## Phase 2: Install Omarchy Layer

Now that you have a secure Arch base, install Omarchy's configuration layer.

### Option A: Official Omarchy Install (Recommended)

The Omarchy install script should work on any Arch base:

```bash
# Download and run Omarchy installer
wget -qO- https://omarchy.org/install | bash

# Or for minimal Omarchy (no Spotify, Pinta, OBS, LocalSend):
# wget -qO- https://omarchy.org/install-bare | bash
```

**What this does:**
- Installs Hyprland and dependencies
- Configures dotfiles
- Installs applications
- Sets up themes
- Configures system services

**What it WON'T do:**
- Repartition your disk (already installed)
- Change your encryption (already configured)
- Modify your boot setup (already working)

### Option B: Manual Omarchy Installation

If you want more control, install Omarchy's components manually:

```bash
# Clone Omarchy repository
git clone https://github.com/basecamp/omarchy.git ~/.local/share/omarchy
cd ~/.local/share/omarchy

# Review the install script first
less install.sh

# Run it
bash install.sh
```

This gives you the chance to review what it's doing before execution.

### Option C: Cherry-Pick Omarchy Components

Install only what you want from Omarchy:

```bash
# 1. Install Hyprland
sudo pacman -S hyprland waybar rofi-wayland dunst kitty swaybg \
    swaylock swayidle grim slurp wl-clipboard \
    xdg-desktop-portal-hyprland

# 2. Clone Omarchy for configs
git clone https://github.com/basecamp/omarchy.git ~/omarchy-source

# 3. Selectively copy configs
cp -r ~/omarchy-source/config/hyprland ~/.config/
cp -r ~/omarchy-source/config/waybar ~/.config/
cp -r ~/omarchy-source/config/kitty ~/.config/
# ... etc

# 4. Install apps you want
sudo pacman -S firefox neovim spotify-launcher
```

---

## Phase 3: Security Hardening Post-Omarchy

After installing Omarchy, you may need to adjust some settings:

### 3.1 Disable Auto-Login (If Enabled)

Omarchy may enable auto-login after disk decryption. If you want two-factor security:

```bash
# Check if auto-login is enabled
cat /etc/systemd/system/getty@tty1.service.d/autologin.conf

# If exists and you don't want it:
sudo rm /etc/systemd/system/getty@tty1.service.d/autologin.conf
sudo systemctl daemon-reload
```

**Trade-off:**
- With auto-login: Single password (LUKS) for convenience
- Without auto-login: Two passwords (LUKS + user) for security

### 3.2 Verify Encryption Still Works

After Omarchy installation:

```bash
# Check encryption status
lsblk -f | grep crypto

# Verify ephemeral swap
sudo cat /etc/crypttab | grep swap
# Should show: swap ... /dev/urandom ...

# Verify ephemeral tmp
sudo cat /etc/crypttab | grep tmp
# Should show: tmp ... /dev/urandom ...

# Reboot and verify swap is re-encrypted
sudo reboot
# After reboot, check swap UUID changed:
lsblk -f | grep swap
```

### 3.3 Review Omarchy's Security Configuration

Omarchy includes security configurations. Review them:

```bash
# Check what Omarchy configured
ls -la ~/.local/share/omarchy/config/

# Review security-related scripts
ls -la ~/.local/share/omarchy/migrations/

# Check for any conflicting configs
diff ~/.bashrc ~/.local/share/omarchy/default/bashrc
```

---

## Potential Issues & Solutions

### Issue 1: Omarchy Expects BTRFS

**Symptom**: Omarchy scripts reference btrfs commands

**Solution**:
```bash
# Edit Omarchy scripts to skip btrfs-specific features
cd ~/.local/share/omarchy
grep -r "btrfs" .

# Comment out or skip btrfs-specific sections
# Your ext4 filesystem works fine, just no snapshots
```

**Alternative**: Use LVM snapshots instead:
```bash
# Create snapshot before major changes
sudo lvcreate -L 5G -s -n root_snapshot /dev/vg-data/lv-cryptroot

# Rollback if needed
sudo lvconvert --merge /dev/vg-data/root_snapshot
```

### Issue 2: Display Manager Conflicts

**Symptom**: Omarchy installs a display manager, conflicts with your setup

**Solution**:
```bash
# Disable Omarchy's display manager if you prefer terminal login
sudo systemctl disable gdm  # or whatever DM it installed
sudo systemctl disable sddm

# Start Hyprland manually from terminal
echo "exec Hyprland" >> ~/.bash_profile
```

### Issue 3: Package Conflicts

**Symptom**: Omarchy tries to install packages that conflict

**Solution**:
```bash
# Before running Omarchy installer, review packages
wget -qO- https://omarchy.org/install > /tmp/omarchy-install.sh
less /tmp/omarchy-install.sh

# Manually run sections, skipping conflicts
# Or edit the script to skip problematic packages
```

---

## Maintenance

### Updating Omarchy

```bash
# Omarchy includes a migration system
cd ~/.local/share/omarchy
git pull

# Run migrations
bash install.sh  # Safe to re-run, skips completed items
```

### Updating Your Security Setup

Your encryption setup is independent of Omarchy:

```bash
# Update kernel
sudo pacman -S linux

# Regenerate initramfs (includes your encrypt+lvm2 hooks)
sudo mkinitcpio -p linux

# Update GRUB if needed
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## Testing Plan

Before committing to this on your main machine:

### Test in VM (Recommended)

1. **Setup VM**:
   - VirtualBox or virt-manager
   - UEFI mode
   - 30GB disk
   - 4GB RAM

2. **Test Process**:
   - Follow Phase 1 (your security setup)
   - Verify encryption works
   - Follow Phase 2 (Omarchy installation)
   - Test Hyprland launches
   - Reboot, verify security features intact

3. **Validation**:
   - Encryption still works after Omarchy ✓
   - Ephemeral swap/tmp still functional ✓
   - Hyprland and apps work ✓
   - No conflicts ✓

---

## Alternative Approaches

### Approach 1: Omarchy First, Then Secure

**Process**:
1. Install Omarchy normally (BTRFS + auto-login)
2. Backup configurations
3. Reinstall with your security setup
4. Restore Omarchy configs

**Pros**: See Omarchy's full setup first
**Cons**: More work, reinstall required

### Approach 2: Omarchy ISO + Manual Partitioning

Some users have had success with:

```bash
# Boot Omarchy ISO
# Press Ctrl+C when installer starts
# Do manual partitioning with your LVM+LUKS setup
# Then run archinstall with --config pointing to your partitions
```

**Pros**: Uses Omarchy's ISO
**Cons**: More complex, less documented

### Approach 3: Fork Omarchy Installer

**Process**:
1. Fork Omarchy repository
2. Modify install scripts for LVM+LUKS
3. Run your custom installer

**Pros**: Full control, repeatable
**Cons**: Maintenance burden, need to sync upstream

---

## Recommended Path (TL;DR)

**For most users wanting security + Omarchy:**

```bash
# 1. Install secure base (2-3 hours)
# Follow your install.md completely
# Verify encryption works perfectly
# Boot into working system

# 2. Install Omarchy layer (30 minutes)
wget -qO- https://omarchy.org/install | bash
# Let it run, install everything

# 3. Security audit (30 minutes)
# Disable auto-login if desired
# Verify encryption still works
# Test ephemeral swap/tmp after reboot

# 4. Customize (ongoing)
# Tweak Omarchy themes/configs
# Add/remove applications
# Enjoy secure + productive system
```

---

## Expected Outcome

After successful hybrid installation:

**Security** (from your approach):
- ✅ LUKS encrypted root with password
- ✅ LUKS encrypted home with key file
- ✅ Ephemeral swap (re-encrypted each boot)
- ✅ Ephemeral tmp (wiped each boot)
- ✅ No hibernation data leakage
- ✅ LVM flexibility

**Productivity** (from Omarchy):
- ✅ Hyprland Wayland compositor
- ✅ Pre-configured keybindings
- ✅ 11 themes ready to use
- ✅ Development tools installed
- ✅ Applications ready (browser, terminal, etc.)
- ✅ Beautiful, modern desktop

**Best of both worlds!**

---

## File System Comparison

### Your Current Approach
```
/ (root)      -> ext4 on /dev/mapper/root
/home         -> ext4 on /dev/mapper/home
/boot         -> FAT32 on /dev/sdb1
swap          -> ephemeral on /dev/mapper/swap
/tmp          -> ephemeral on /dev/mapper/tmp
```

### After Hybrid Installation
```
# Same as above, unchanged
# Omarchy's configs live in /home/user/.config/
# Omarchy's apps installed via pacman
# No filesystem changes needed
```

---

## Summary

**This hybrid approach gives you:**

1. **Security-first foundation**: Your proven LUKS on LVM setup
2. **Productivity layer**: Omarchy's beautiful Hyprland environment
3. **Flexibility**: Pick which Omarchy components you want
4. **Control**: Keep your security preferences
5. **Modern**: Get latest apps and configs from Omarchy
6. **Maintained**: Benefit from Omarchy's updates

**Trade-offs to accept:**

- ⚠️ No BTRFS snapshots (use LVM snapshots instead)
- ⚠️ Some Omarchy features may expect BTRFS (can be worked around)
- ⚠️ More initial setup time than pure Omarchy
- ⚠️ Need to maintain both layers

**Is it worth it?**

**Yes, if:** Security is critical, you want ephemeral swap/tmp, but also want a beautiful working environment immediately.

**No, if:** You trust Omarchy's security model and want zero configuration time.

---

## Next Steps

1. **Decide on approach**: Full hybrid, or cherry-pick components?
2. **Test in VM first**: Validate the process
3. **Document your specific steps**: Create your own guide
4. **Share findings**: Help others with this hybrid approach
5. **Contribute back**: If you find improvements, share with community

Good luck with your secure + productive Arch installation! 🔒✨
