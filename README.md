# Arch Linux Install Guide

A complete, self-contained guide for installing Arch Linux, covering two approaches:

- **Manual installation** — partitioning, formatting, `pacstrap`, chroot, and bootloader setup done by hand, step by step.
- **`archinstall`** — the official guided/automated installer.

See [`CONFIG.md`](./CONFIG.md) for the full step-by-step instructions and configuration details.

## Requirements

- A USB drive (2GB+) and a way to write the ISO to it (`dd`, [Rufus](https://rufus.ie/), Ventoy, etc.)
- The latest [Arch Linux ISO](https://archlinux.org/download/)
- A machine that supports UEFI boot (this guide assumes UEFI/GPT — legacy BIOS/MBR needs minor adjustments, noted in `CONFIG.md`)
- An active internet connection during install (wired or Wi-Fi)
- Basic familiarity with the Linux command line
- Willingness to back up any existing data — partitioning will erase the target disk

## Structure

```
.
├── README.md   # this file — overview and requirements
└── CONFIG.md    # full install steps for both methods
```

## References

- [Official Arch install guide](https://wiki.archlinux.org/title/Installation_guide)
- [archinstall documentation](https://wiki.archlinux.org/title/Archinstall)
