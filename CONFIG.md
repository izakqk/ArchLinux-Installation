# Installing Arch Linux: A Beginner's Guide

Arch Linux has a reputation for being intimidating, but you've got two real paths in: the modern guided installer (`archinstall`), which has made it genuinely approachable for a first-timer, or the full manual installation, which teaches you exactly what's happening under the hood. This guide covers both, step by step.

## Phase 1: Get the Installer Onto a USB Drive

1. **Download the ISO** from the [official Arch Linux downloads page](https://archlinux.org/download/). Grab the latest release — Arch is a rolling release, so there's no "version number" to worry about, just get the current image.
2. **Verify the download (optional but good practice)**. The download page lists a checksum you can compare against your file using `sha256sum` on Linux/macOS, or a tool like `CertUtil` on Windows. This confirms the file wasn't corrupted or tampered with. # You can skip it
3. **Flash the ISO to your USB drive** using [Rufus](https://rufus.ie/) (Windows) or [BalenaEtcher](https://www.balena.io/etcher/) (Windows/macOS/Linux). This process erases everything on the USB drive, so back up anything on it first.

## Phase 2: Boot Into the Installer

1. **Disable Secure Boot** in your BIOS/UEFI settings. Restart your PC and spam a key like `F2`, `F10`, `F12`, or `Del` right as it powers on to enter the BIOS menu — the exact key depends on your motherboard/laptop brand.
2. **Boot from the USB drive.** Most systems let you pick a one-time boot device with a key like `F12` or `Esc` during startup, without needing to permanently change your boot order.
3. Once it boots, select the first menu option (the Arch Linux install medium) and you'll land in a terminal prompt, logged in as `root`.

## Phase 3: Live Environment Setup

- **Set your console keyboard layout** if you're not on a US layout (skip this if you are):
  ```
  loadkeys de
  ```
  (Replace `de` with your layout code — run `localectl list-keymaps` to browse options.)

## Phase 4: Connect to the Internet

- **Ethernet** usually just works via DHCP — no setup needed.
- **Wi-Fi** needs to be configured manually with `iwctl`:
  ```
  iwctl
  device list
  station wlan0 scan
  station wlan0 get-networks
  station wlan0 connect "Your-Network-Name"
  exit
  ```
  (Swap `wlan0` for whatever your device is actually called — check with `device list`.)
- **Confirm you're online** with `ping archlinux.org`. If you see replies, you're good to go.
- **Sync the system clock** (needed for package signature verification to work correctly):
  ```
  timedatectl set-ntp true
  ```

## Phase 5: Install the System

### The Easy Way: `archinstall`

This is the officially supported guided installer and the recommended starting point for newcomers. Just type:

```
archinstall
```

You'll be walked through a menu-driven setup covering:

- **Language/keyboard layout** — pick your locale
- **Mirror region** — choosing your country speeds up downloads noticeably
- **Disk configuration** — "best-effort default partition layout" is the safe, simple choice; make sure you pick the correct drive, since this step erases it. archinstall will lay out roughly the same EFI/swap/root/home split covered in the manual method below — it just does it for you
- **Filesystem** — ext4 is the safe default; Btrfs if you want snapshots later
- **Bootloader** — systemd-boot (simpler, UEFI-only) or GRUB (more universal, supports older BIOS and dual-boot scenarios)
- **Profile** — choose "Desktop" and then a desktop environment (GNOME and KDE Plasma are the most beginner-friendly; Hyprland/Sway if you want a tiling window manager and don't mind a steeper learning curve)
- **Graphics drivers** — pick the one matching your GPU (open-source `mesa` drivers work well for AMD/Intel; Nvidia users generally want the proprietary driver option)
- **Network configuration** — "Copy ISO network configuration" carries over the Wi-Fi setup you just did, so you don't have to redo it
- **User account** — set a root password and create your own user with sudo access (you'll use this account day-to-day, not root)

When everything's configured, select **Install** and let it run.

### The Manual Way: Full Control

This is the "traditional" Arch installation people mean when they joke about "I use Arch, btw." It takes longer and has more room for mistakes, but you'll understand exactly what's on your system by the end.

**Check your firmware type first:**

```
ls /sys/firmware/efi/efivars
```

If that directory exists, you're on **UEFI** — follow the main steps below. If you get "No such file or directory," you're on legacy **BIOS**, and you'll partition differently (see the callout below).

**1. Identify your disk**

```
lsblk
```

Find your target drive's name (e.g. `/dev/sda` or `/dev/nvme0n1`). Double-check this — the next step erases it.

**2. Partition the disk**

We'll use a **4-partition layout** — this is the same shape whether you're single-booting or dual-booting, it just shifts depending on what's already on the disk:

| Partition | Purpose |
|---|---|
| **EFI System Partition** | Lets your firmware find and boot the OS |
| **Swap Partition** | Backup memory for when RAM runs out (also enables hibernation) |
| **Root Partition** | The OS itself — system files and installed packages |
| **Home Partition** | Your personal files, configs, and downloads, kept separate from the OS |

Keeping `/home` on its own partition means you can reinstall or distro-hop later by wiping just the root partition, without touching your personal files.

**If you're doing a single-boot UEFI install** (no other OS on this disk):

```
cfdisk /dev/sdX
```

Create a **GPT** layout with four partitions:
- **EFI System Partition** — 512MB–1GB, type "EFI System"
- **Swap Partition** — sized to match your RAM (e.g. 8GB if you have 8GB RAM; less critical if you have 16GB+ and don't need hibernation), type "Linux swap"
- **Root Partition** — 40–100GB is plenty for the OS and packages, type "Linux filesystem"
- **Home Partition** — the rest of the disk, type "Linux filesystem"

> **Legacy BIOS instead?** BIOS systems use an **MBR** partition table, not GPT, and don't need an EFI partition at all. In `cfdisk`, select the `dos` label instead of `gpt`, then create a swap, root, and home partition (same sizing as above) plus a small unformatted 1MB "BIOS boot" partition if you plan to use GRUB. Skip the `mkfs.fat`/`/boot` mount below — there's no EFI partition to format or mount.

> **Dual-booting with Windows?** Don't repartition or format anything Windows already made. Boot into Windows first and shrink its partition using Disk Management to free up space, *then* boot the Arch ISO. In `cfdisk`, you'll see Windows's existing partitions (a small EFI System Partition it already created, plus its main NTFS partition) — leave those alone. In the freed-up unallocated space, create just **three** new partitions: swap, root, and home (same sizing as above). You do **not** need a second EFI partition — both operating systems share the one Windows already made.

**3. Format the partitions**

For single-boot (assuming `sdX1`=EFI, `sdX2`=swap, `sdX3`=root, `sdX4`=home):

```
mkfs.fat -F 32 /dev/sdX1
mkswap /dev/sdX2
mkfs.ext4 /dev/sdX3
mkfs.ext4 /dev/sdX4
```

For dual-boot, skip the `mkfs.fat` line entirely — Windows's existing EFI partition is already formatted and should stay that way. Just format your new swap, root, and home partitions (adjust the numbers to match whatever `cfdisk`/`lsblk` shows for your new partitions).

**4. Mount the partitions and enable swap**

Single-boot:

```
mount /dev/sdX3 /mnt
mount --mkdir /dev/sdX1 /mnt/boot
mount --mkdir /dev/sdX4 /mnt/home
swapon /dev/sdX2
```

Dual-boot: mount *Windows's existing* EFI partition at `/mnt/boot` instead of a new one — that's what lets one GRUB installation manage both OSes:

```
mount /dev/sdX3 /mnt
mount --mkdir /dev/sdX1 /mnt/boot    # this is Windows's existing EFI partition
mount --mkdir /dev/sdX4 /mnt/home
swapon /dev/sdX2
```

**5. (Optional but recommended) Update your mirror list**

Faster, more reliable package downloads — especially useful since `pacstrap` is about to pull a lot of packages:

```
pacman -S reflector --noconfirm
reflector --country YourCountry --sort rate --save /etc/pacman.d/mirrorlist
```

**6. Install the base system**

```
pacstrap -K /mnt base linux linux-firmware nano networkmanager sudo
```

If you're on an Intel or AMD CPU, also add the matching microcode package — it patches CPU-level bugs and is worth including from the start:

```
pacstrap -K /mnt intel-ucode    # Intel CPU
pacstrap -K /mnt amd-ucode      # AMD CPU
```

This pulls the kernel, firmware, and core utilities from the mirrors — it's the step that actually takes a while. Don't forget `linux` and `linux-firmware` here; a surprising number of people miss them and end up with an unbootable system.

**7. Generate the filesystem table**

```
genfstab -U /mnt >> /mnt/etc/fstab
```

This also picks up your swap partition automatically, since it's already mounted with `swapon`.

**8. Chroot into the new system**

```
arch-chroot /mnt
```

Everything from here on happens *inside* your new install, not the live USB environment.

**9. Basic system configuration**

```
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

Edit `/etc/locale.gen`, uncomment your locale (e.g. `en_US.UTF-8 UTF-8`), then run `locale-gen`. Create `/etc/locale.conf` with `LANG=en_US.UTF-8` inside it.

If you set a non-US keyboard layout back in Phase 3, make it persist after reboot by creating `/etc/vconsole.conf` with `KEYMAP=de` (or whatever code you used) inside it.

Set a hostname: `echo myhostname > /etc/hostname`. Then add matching entries to `/etc/hosts` so the system resolves its own name correctly:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   myhostname.localdomain   myhostname
```

**10. Set the root password and create a user**

```
passwd
useradd -m -G wheel -s /bin/bash yourusername
passwd yourusername
```

Enable sudo for that user by running `EDITOR=nano visudo` and uncommenting the `%wheel ALL=(ALL:ALL) ALL` line.

**11. Install and configure the bootloader (GRUB)**

For **UEFI, single-boot**:

```
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

For **legacy BIOS**:

```
pacman -S grub
grub-install --target=i386-pc /dev/sdX
grub-mkconfig -o /boot/grub/grub.cfg
```

(Note: for BIOS, point `grub-install` at the whole disk, e.g. `/dev/sda` — not a partition number.)

For **dual-boot with Windows** (UEFI), do the same UEFI install above, then also install `os-prober` so GRUB detects Windows:

```
pacman -S grub efibootmgr os-prober
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

Edit `/etc/default/grub` and uncomment/add this line:

```
GRUB_DISABLE_OS_PROBER=false
```

Then generate the config:

```
grub-mkconfig -o /boot/grub/grub.cfg
```

You should see a line like `Found Windows Boot Manager on ...` in the output — if you don't, double-check that Windows's EFI partition is mounted at `/boot` (Step 4) before running this.

**12. Enable networking, then exit and reboot**

```
systemctl enable NetworkManager
exit
```

Back in the live environment, unmount and reboot:

```
umount -R /mnt
reboot
```

Both methods land you at the same place — `archinstall` just automates steps 1–11 for you. (Note: `archinstall`'s guided flow can also detect an existing Windows install and offer a dual-boot-safe partition layout, if you'd rather not do this part by hand.)

## Phase 6: First Boot

1. Once installation finishes, type `reboot`.
2. Remove the USB drive during the restart so the machine boots from your internal disk instead.
3. Log in with the user account you created.

## After You're In

A few things worth doing early on:

- Run `sudo pacman -Syu` to make sure everything's fully up to date
- Install an AUR helper like `yay` or `paru` — the AUR (Arch User Repository) is where most extra software lives
- Read the [Arch Wiki](https://wiki.archlinux.org) as things come up. It's genuinely one of the best pieces of Linux documentation that exists, distro-agnostic tips included

## A Word on Arch's Philosophy

Arch doesn't hold your hand after install — there's no automatic "make it pretty" step. You'll be doing a lot of your own configuration (network, audio, a desktop environment if you skipped that step, etc.). This is intentional: Arch's whole philosophy is that you understand and choose what's on your system, rather than inheriting a bunch of defaults you didn't ask for. It's more work upfront, but it means you'll actually understand your own machine by the end.

## A Note on Dual-Boot Order

Install **Windows first, then Arch** if you're setting up both from scratch. Windows installers tend to overwrite bootloaders and boot entries without asking, so installing it second will likely knock out your Arch boot entry. If Windows is already installed and you're adding Arch afterward (the more common case), you're fine — just follow the dual-boot notes above. If Windows ever does overwrite GRUB after an update, boot the Arch ISO again, `arch-chroot` back in, and rerun the `grub-install`/`grub-mkconfig` steps from Step 11 to restore it.
