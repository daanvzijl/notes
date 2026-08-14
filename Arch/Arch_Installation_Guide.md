# Arch Linux Installation & Hyprland Setup Guide

This guide provides step-by-step instructions for installing Arch Linux with a **BTRFS subvolume layout**, **Linux Zen Kernel**, **GRUB with Snapper snapshots**, and the **Hyprland** desktop environment.

---

## Table of Contents

1. [Pre-Installation Setup (Live ISO & SSH)](#1-pre-installation-setup-live-iso--ssh)
2. [Disk Partitioning & BTRFS Subvolumes](#2-disk-partitioning--btrfs-subvolumes)
3. [Base System Installation](#3-base-system-installation)
4. [System Configuration (Chroot)](#4-system-configuration-chroot)
5. [Consolidated System Package Installation](#5-consolidated-system-package-installation)
6. [First Reboot & TTY Shell Login](#6-first-reboot--tty-shell-login)
7. [Post-Reboot Setup (AUR Packages & Services)](#7-post-reboot-setup-aur-packages--services)
8. [Dotfiles & Reference Configurations](#8-dotfiles--reference-configurations)
9. [Optional: LAVD Scheduler](#9-optional-lavd-scheduler)
10. [Optional: Gaming Tools](#10-optional-gaming-tools)

---

## 1. Pre-Installation Setup (Live ISO & SSH)

Boot into the Arch Linux Live ISO and complete the initial setup:

```bash
# 1. Set root password for the live environment (allows SSH access)
passwd

# 2. Start SSH daemon to allow remote installation from another machine (optional)
systemctl start sshd

# 3. Set timezone (replace Europe/Amsterdam with your timezone)
timedatectl set-timezone Europe/Amsterdam

# 4. Enable Network Time Protocol (NTP)
timedatectl set-ntp true
```

> **Note**: Setting `passwd` on the live ISO establishes a root password so you can start `sshd` and connect to the installer via SSH from another machine.

---

## 2. Disk Partitioning & BTRFS Subvolumes

> **Note**: Replace `/dev/nvme0n1` with your actual drive path (e.g. `/dev/sda`).

### 2.1 Create Partitions
Using `cfdisk` or `fdisk`:
- Partition 1: **EFI System Partition** (512 MB, type `EFI System`)
- Partition 2: **Linux Filesystem** (Remaining space, type `Linux filesystem`)

```bash
fdisk /dev/nvme0n1
```
*Partition Table layout: GPT (`g`), 1st partition `+512M` (type EFI), 2nd partition rest of disk (type Linux filesystem).*

### 2.2 Format Partitions
```bash
# Format EFI partition as FAT32
mkfs.fat -F 32 /dev/nvme0n1p1

# Format Root partition as BTRFS
mkfs.btrfs /dev/nvme0n1p2
```

### 2.3 Create BTRFS Subvolumes
```bash
# Mount root filesystem temporarily
mount /dev/nvme0n1p2 /mnt

# Create subvolumes
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@srv
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@game

# Unmount temporary mount
umount /mnt
```

### 2.4 Mount Subvolumes & EFI
Mount options: `noatime,ssd,discard=async`

```bash
# Mount root subvolume
mount -o noatime,ssd,discard=async,subvol=@ /dev/nvme0n1p2 /mnt

# Create mount point directories
mkdir -p /mnt/{home,root,srv,efi}
mkdir -p /mnt/var/{cache,tmp,log}
mkdir -p /mnt/opt/game

# Mount remaining subvolumes
mount -o noatime,ssd,discard=async,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o noatime,ssd,discard=async,subvol=@root /dev/nvme0n1p2 /mnt/root
mount -o noatime,ssd,discard=async,subvol=@srv /dev/nvme0n1p2 /mnt/srv
mount -o noatime,ssd,discard=async,subvol=@cache /dev/nvme0n1p2 /mnt/var/cache
mount -o noatime,ssd,discard=async,subvol=@tmp /dev/nvme0n1p2 /mnt/var/tmp
mount -o noatime,ssd,discard=async,subvol=@log /dev/nvme0n1p2 /mnt/var/log
mount -o noatime,ssd,discard=async,subvol=@game /dev/nvme0n1p2 /mnt/opt/game

# Mount EFI partition
mount /dev/nvme0n1p1 /mnt/efi

# Verify mounts
findmnt -R /mnt
```

---

## 3. Base System Installation

Install the essential base system packages using `pacstrap`:
Remember to swap `intel-ucode` for `amd-ucode` if on an AMD CPU.

```bash
pacstrap -K /mnt \
    base \
    linux-zen \
    linux-zen-headers \
    linux-firmware \
    intel-ucode \
    btrfs-progs \
    grub \
    efibootmgr \
    networkmanager \
    sudo \
    vim
```

Generate the file system table (`fstab`):
```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

---

## 4. System Configuration (Chroot)

Chroot into the installed system:
```bash
arch-chroot /mnt
```

### 4.1 Locale & Keyboard
Edit `/etc/locale.gen` and uncomment both `en_US.UTF-8 UTF-8` (system language) and `en_GB.UTF-8 UTF-8` (for EU-styled 24-hour clocks and `DD/MM/YY` date formatting), then generate locales:

```bash
# Generate enabled locales
locale-gen

# Set primary system language
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set European-styled time/date formatting (24h clock, DD/MM/YYYY)
echo "LC_TIME=en_GB.UTF-8" >> /etc/locale.conf

# Set console keyboard mapping
echo "KEYMAP=us" > /etc/vconsole.conf
```

### 4.2 Hostname & Users
```bash
# Set hostname (replace t15 with your desired hostname)
echo "t15" > /etc/hostname

# Set root password
passwd

# Create user and add to wheel group
useradd -m -G wheel daan
passwd daan

# Configure sudoers
EDITOR=vim visudo
```
*In `visudo`, uncomment `%wheel ALL=(ALL:ALL) ALL`.*

### 4.3 Optimize Mirrors
Install `reflector` and generate an updated mirrorlist. **Be sure to edit the command and replace `Netherlands` with your own country** (e.g., `"United States"`, `"Germany"`, etc.):

```bash
pacman -S reflector rsync

# Replace 'Netherlands' with your country
reflector -c Netherlands -a 12 --sort rate --save /etc/pacman.d/mirrorlist
```

Edit the default reflector config to your liking if you enable the timer job later in this guide (I do).
`/etc/xdg/reflector/reflector.conf`

I personally only set/change the following keys/values:
```bash
--country Netherlands
--sort score
```

---

## 5. Consolidated System Package Installation

All system packages consolidated into this single installation step before first reboot:

```bash
pacman -Syu \
    base-devel \
    mtools \
    network-manager-applet \
    blueman \
    openssh \
    git \
    acpid \
    snapper \
    grub-btrfs \
    inotify-tools \
    rtkit \
    snap-pac \
    hyprland \
    uwsm \
    libnewt \
    ly \
    waybar \
    hyprpaper \
    hyprlock \
    hypridle \
    hyprpolkitagent \
    mako \
    wofi \
    xdg-user-dirs \
    thunar \
    alacritty \
    slurp \
    grim \
    wl-clipboard \
    xdg-desktop-portal \
    xdg-desktop-portal-hyprland \
    qt5-wayland \
    qt6-wayland \
    pipewire \
    pipewire-pulse \
    pipewire-alsa \
    pipewire-jack \
    wireplumber \
    sof-firmware \
    bluez \
    bluez-utils \
    brightnessctl \
    playerctl \
    udiskie \
    power-profiles-daemon \
    gnome-keyring \
    gnome-themes-extra \
    cmake \
    noto-fonts \
    noto-fonts-cjk \
    noto-fonts-emoji \
    ttf-jetbrains-mono-nerd \
    ttf-font-awesome \
    man-db \
    man-pages \
    texinfo \
    firefox
```

### 5.1 Optional: GPU Drivers

Install graphics drivers if needed:

```bash
# AMD (discrete or integrated)
pacman -S mesa vulkan-radeon libva-mesa-driver

# NVIDIA — Turing (RTX 20xx) and newer
# Note: Hyprland has no official Nvidia support. Works for many, but expect more
# friction than AMD/Intel (XWayland sync issues, occasional driver/kernel breakage).
# See: https://wiki.hypr.land/Nvidia/
pacman -S nvidia-open-dkms nvidia-utils vulkan-icd-loader

# NVIDIA — GTX 900/1000 series (Maxwell/Pascal): use legacy driver from AUR instead, official repos no longer support these cards
```

Further configuration for NVIDIA:

Enable DRM modesetting (required for Wayland/Hyprland on Nvidia)
```bash
echo "options nvidia_drm modeset=1" > /etc/modprobe.d/nvidia.conf
```

Add Nvidia modules to initramfs for early KMS — edit /etc/mkinitcpio.conf,
add to the MODULES=() array: nvidia nvidia_modeset nvidia_uvm nvidia_drm


### 5.2 Bootloader Configuration & Enable System Services
```bash
# Rebuild initramfs
mkinitcpio -P

# Install GRUB for EFI
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --recheck

# Generate GRUB configuration
grub-mkconfig -o /boot/grub/grub.cfg

# Enable core systemd services
systemctl enable \
    NetworkManager.service \
    bluetooth.service \
    ly@tty1.service \
    acpid.service \
    sshd.service \
    systemd-timesyncd.service \
    reflector.timer \
    power-profiles-daemon.service
```

---

## 6. First Reboot & TTY Shell Login

Exit the chroot environment, unmount the filesystems, and reboot into your new system:

```bash
# Exit chroot environment
exit

# Unmount all mounted subvolumes and partitions
umount -R /mnt

# Reboot into installed system
reboot
```

### Logging into the Shell
1. Upon system reboot, the `ly` TUI display manager will launch on **TTY1**.
2. **Log into the shell** (TTY console) as user `daan`.
3. *(Optional)* If you prefer working remotely, connect via **SSH** again from another machine (`sshd.service` is active).

> **Why log into the TTY shell first?**
> We do not launch Hyprland immediately because we must install AUR packages (`paru`, `wayfreeze-git`, `wleave`), enable user-level systemd services, and ensure our terminal (`alacritty`) is correctly configured in `~/.config/hypr/hyprland.lua`.

---

## 7. Post-Reboot Setup (AUR Packages, Services & Snapper)

Once logged into your user account shell (`daan`):

### 7.1 Configure Snapper and enable relevant services.

Create snapper root config with:

```bash
sudo snapper -c root create-config /
```
And enable relevant services:

```bash
sudo systemctl enable --now \
    grub-btrfsd.service \
    snapper-cleanup.timer
```

### 7.2 Update System & Install Paru (AUR Helper)
```bash
sudo pacman -Syu

# Clone and compile paru
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
cd ..
rm -rf paru
```

### 7.3 Install AUR Packages (`wayfreeze` & `wleave`)
```bash
# Install wayfreeze (screen freezing for grim/slurp screenshots) and wleave (Wayland logout menu)
paru -S wayfreeze-git wleave
```

### 7.4 Enable User Services
```bash
systemctl --user enable --now \
    hyprpaper.service \
    hyprpolkitagent.service \
    waybar.service \
    xdg-user-dirs.service \
    hypridle.service
```

### 7.5 Update Hyprland Plugins & Verification
```bash
# Update Hyprland plugin manager
hyprpm update
```

### 7.6 Snapper Configuration
You may want to inspect your Snapper configuration (`/etc/snapper/configs/root`) to review snapshot retention settings according to your needs.

---

## 8. Dotfiles & Reference Configurations

> Reference Note: The configuration files located in the [`files/`](./files/) directory are my personal dotfiles and are provided purely for reference as examples of how I configure each application in this setup.

### 8.1 Reference Files
- **Hyprland**: [`files/hypr/hyprland.lua`](./files/hypr/hyprland.lua)
- **Hypridle**: [`files/hypr/hypridle.conf`](./files/hypr/hypridle.conf)
- **Hyprlock**: [`files/hypr/hyprlock.conf`](./files/hypr/hyprlock.conf)
- **Hyprpaper**: [`files/hypr/hyprpaper.conf`](./files/hypr/hyprpaper.conf)
- **Waybar**: [`files/waybar/config.jsonc`](./files/waybar/config.jsonc) & [`files/waybar/style.css`](./files/waybar/style.css)
- **Mako**: [`files/mako/config`](./files/mako/config)

### 8.2 Custom Keybindings & Input Adjustments

Notable custom keybindings and input overrides in [`files/hypr/hyprland.lua`](./files/hypr/hyprland.lua):

- **Area Screenshot (`Super + Shift + S`)**: Uses `wayfreeze` to freeze the frame, `grim` + `slurp` to select a region, and copies the screenshot directly to the clipboard (`wl-copy`).
- **Lock Screen (`Super + L`)**: Executes `hyprlock`.
- **Caps Lock Modifier (`kb_options = "caps:ctrl_modifier"`)**: Remaps Caps Lock to behave as Ctrl.
- **Flat Mouse Profile (`accel_profile = "flat"`)**: Disables mouse acceleration for flat, raw pointer input.

### 8.3 Starting the Desktop Environment
Once your system configuration is complete, log in using the `ly` display manager interface on TTY1 and select the **UWSM-managed Hyprland** session to start your desktop environment.


## 9. Optional: LAVD Scheduler

This step is optional, but I personally like to enable/load the LAVD scheduler. This can be done with eBPF.

First install:
```bash
sudo pacman -Syu scx-scheds scx-tools
```

Then create/edit the following file: `/etc/scx_loader/config.toml`

```toml
default_sched = "scx_lavd"
default_mode = "Auto"

[scheds.scx_lavd]
auto_mode = ["--performance"]
```
Enable the scx_loader service:

```bash
sudo systemctl enable --now scx_loader
```

## 10. Optional: Gaming Tools

Requires the multilib repository enabled in `/etc/pacman.conf` (uncomment the `[multilib]` section and the `Include` line beneath it, then `sudo pacman -Syu`) — needed for `steam`, `lib32-gamemode`, and `lib32-mangohud`.

```bash
pacman -S steam gamemode lib32-gamemode mangohud lib32-mangohud
```

Add yourself to the gamemode group (required for gamemode to work):

```bash
sudo groupadd -f gamemode
sudo usermod -aG gamemode daan
```

Reboot or re-login for the group change to take effect.
