# Arch Linux Installation & Configuration

Personal documentation for secure Arch Linux installations with advanced encryption.

## Available Guides

### Installation Approaches

1. **[Manual Installation](installation/install.md)** - Security-first, step-by-step manual installation
   - LUKS on LVM with ephemeral swap/tmp
   - Maximum security and transparency
   - Best for learning and custom requirements

2. **[Hybrid: Omarchy + Security](installation/hybrid-omarchy-security.md)** - NEW! 🎯
   - Combine your security approach with Omarchy's productivity
   - Best of both worlds: ephemeral swap/tmp + Hyprland/apps
   - Secure foundation, productive environment

### Analysis & Comparison

- **[Comparison: Manual vs Omarchy](COMPARISON.md)** - Detailed comparison of approaches
- **[Proposed Improvements](PROPOSED_IMPROVEMENTS.md)** - Roadmap for repository enhancements

## Quick Decision Guide

**Choose Manual Installation if:**
- You want to learn Linux deeply
- Security is paramount (ephemeral swap/tmp required)
- You need custom configurations
- You prefer complete control

**Choose Hybrid Approach if:**
- You want security + productivity
- You like Omarchy's Hyprland setup
- You want ephemeral swap/tmp but don't want to configure everything
- You're willing to do two-phase installation

**Choose Pure Omarchy if:**
- Speed is priority (5-30 min setup)
- You trust their security model
- You want zero configuration
- Standard LUKS + BTRFS is sufficient

## Features

- **Advanced Encryption**: LUKS on LVM with ephemeral swap/tmp
- **Security-First**: Prevents data leakage, password-protected boot
- **Well-Documented**: Step-by-step with rationale
- **Hybrid Option**: Now supports Omarchy integration!

## Last Updated

2025-11-08