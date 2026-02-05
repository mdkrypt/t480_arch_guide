# ThinkPad T480 Arch Linux Installation Guide 

[<img src="https://img.shields.io/badge/Hardware-ThinkPad_T480-red?style=for-the-badge&logo=lenovo"/>](https://lenovo.com)
[<img src="https://img.shields.io/badge/OS-Arch_Linux-1793d1?style=for-the-badge&logo=arch-linux"/>](https://archlinux.org)
[<img src="https://img.shields.io/badge/Boot-GRUB_Btrfs-blue?style=for-the-badge&logo=gnu"/>](https://www.gnu.org/software/grub/)
[<img src="https://img.shields.io/badge/Compositor-Hyprland-orange?style=for-the-badge&logo=wayland"/>](https://hyprland.org)
[<img src="https://img.shields.io/badge/Desktop-KDE_Plasma-1d99f3?style=for-the-badge&logo=kde"/>](https://kde.org/plasma-desktop/)
[<img src="https://img.shields.io/badge/GPU-MX150_D3COLD-grey?style=for-the-badge&logo=nvidia"/>](https://nvidia.com)

A specialized installation guide for the **ThinkPad T480**. Optimized for **Btrfs Snapshots**, **KDE Plasma**, **Hyprland**, **LUKS2**, **0W Nvidia Power Draw**, and **Update Resiliency**.

---

## Prerequisites

**Required Knowledge:**
* Basic Linux terminal knowledge and command-line navigation
* Familiarity with text editors (`vim` or `nano`)
* Understanding of disk partitions and filesystem concepts
* Comfortable with command-line package management
* Ability to recover from boot failures (BIOS access, boot media creation)

**Required Hardware:**
* **Laptop:** ThinkPad T480
* **GPU:** Nvidia MX150 (Optional - Guide includes steps to fully disable it for battery life. Intel-only models can skip Phase 6.1)
* **Storage:** 1TB NVMe SSD (Guide logic uses `-100G` over-provisioning strategy)
* **Memory:** 32GB RAM (Swapfile calculated at 36GB for safe hibernation)
* **WiFi:** Intel AX210 (Guide includes specific `iwlwifi` firmware and power fixes)
* **Install Media:** USB flash drive (4GB+)

**Recommended:**
* Backup of any existing data (**Installation performs Secure Erase**)
* AC power adapter (Do not install on battery)
* Wired ethernet connection (Optional; WiFi setup included in Phase 1)
* Second device for referencing this guide

**Installation Time Estimate:**
* Experienced users: 2-3 hours
* First-time Arch installers: 4-6 hours

---

## Table of Contents

* [Phase 1: Hardware Prep & Security Wipe](#phase-1)
* [Phase 2: Disk Layout & Encryption](#phase-2)
* [Phase 3: Core System Bootstrap](#phase-3)
* [Phase 4: Bootloader & Disaster Recovery](#phase-4)
* [Phase 5: Desktop & Connectivity Stack](#phase-5)
* [Phase 6: Power, Hardware & Resiliency](#phase-6)
* [Phase 7: Validation & Maintenance](#phase-7)

---

## Phase 1: Hardware Prep & Security Wipe <a name="phase-1"></a>

**Objective:** Configure BIOS and factory-reset the SSD.

### 1. BIOS Optimization

Press **F1** at startup.

* **Secure Boot:** Disabled
* **UEFI:** Only
* **Thunderbolt 3 Assist:** **Disabled** (Critical for C-States/Power)

### 2. Networking Setup

Boot Arch ISO.

```bash
# Increase the font size that is often too small
setfont ter-124b

# Replace "Your_SSID" and "Your_Password" with your actual WiFi credentials
iwctl --passphrase "Your_Password" station wlan0 connect "Your_SSID"
sed -i 's/#ParallelDownloads = 5/ParallelDownloads = 10/' /etc/pacman.conf

```

### 3. Secure Erase

```bash
lsblk # Verify nvme0n1
blkdiscard -f /dev/nvme0n1

```

---

## Phase 2: Disk Layout & Encryption <a name="phase-2"></a>

**Objective:** Single encrypted container holding OS, Data, and Swapfile.

### 1. Partitioning

```bash
gdisk /dev/nvme0n1
# p1: +1G (ef00) - EFI
# p2: Remaining -100G (8300) - Linux LUKS

```

* **`o` → `y**`: New GPT table.
* **`n` → `1` → `Enter` → `+1024M` → `ef00**`: EFI Partition (1GB).
* **`n` → `2` → `Enter` → `-100G` → `8300**`: Linux LUKS Partition (All Remaining Space - 100GB for allowing automatic cleanup).
* **`w` → `y**`: Write and Exit.

### 2. Encryption (LUKS2 + PBKDF2)

**Critical:** Use `pbkdf2` to ensure GRUB can decrypt the drive to read the kernel.

**Security Note:** Deliberately leave 100GB unallocated at the end of the drive 
for over-provisioning. This provides two benefits:
1. **Performance:** NVMe controller uses unallocated space for garbage collection
2. **Security:** Without TRIM/discard, the encrypted partition appears as uniform 
   random data with no metadata leakage about actual space usage.

Do NOT use `:allow-discards` in the cryptdevice parameter in order to maintain this 
security property.

```bash
cryptsetup luksFormat --type luks2 --pbkdf pbkdf2 /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 cryptroot

```

### 3. Formatting & Subvolumes

**Note:** Use `compress=zstd:1` for maximum throughput and lower CPU usage.

```bash
# Format
mkfs.fat -F32 -n BOOT /dev/nvme0n1p1
mkfs.btrfs -L arch /dev/mapper/cryptroot

# Create Subvolumes (Flat Layout)
mount /dev/mapper/cryptroot /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
umount /mnt

# Mount (Mount EFI to /efi for kernel rollback support)
mount -o noatime,compress=zstd:1,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{efi,home,.snapshots,swap}
mount -o noatime,compress=zstd:1,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=zstd:1,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o noatime,subvol=@swap /dev/mapper/cryptroot /mnt/swap
mount /dev/nvme0n1p1 /mnt/efi

```

### 4. Swapfile (Encrypted)

**Adjustment:** 36GB size to guarantee hibernation safety for 32GB RAM.

```bash
truncate -s 0 /mnt/swap/swapfile
chattr +C /mnt/swap/swapfile
dd if=/dev/zero of=/mnt/swap/swapfile bs=1G count=36 status=progress
chmod 600 /mnt/swap/swapfile
mkswap /mnt/swap/swapfile

```

---

## Phase 3: Core System Bootstrap <a name="phase-3"></a>

**Objective:** Install Base OS

### 1. Pacstrap

```bash
# Note: Though powertop and tlp are both installed powertop service must not be started

# Core System & Kernel
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux-firmware \
    intel-ucode intel-media-driver intel-gpu-tools libva-utils \
    sof-firmware btrfs-progs

# Boot & Recovery
pacstrap -K /mnt grub efibootmgr grub-btrfs snapper snap-pac inotify-tools

# Connectivity & Security
pacstrap -K /mnt networkmanager firewalld bluez bluez-utils

# Power Management & Firmware
pacstrap -K /mnt tlp tlp-rdw throttled powertop fwupd

# Tools & Shells (User Preferences)
pacstrap -K /mnt git vim neovim rsync

# Basic Fonts (For Console/TTY)
pacstrap -K /mnt terminus-font otf-font-awesome ttf-font-awesome ttf-fira-sans \
    ttf-firacode-nerd ttf-nerd-fonts-symbols noto-fonts-cjk noto-fonts-emoji \
    ttf-jetbrains-mono ttf-cascadia-code-nerd ttf-iosevka-nerd
```

### 2. Fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
sed -i '/swapfile/d' /mnt/etc/fstab
echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab

```

### 3. Chroot & Basic Config

```bash
arch-chroot /mnt

# 1. Timezone & Clock
# Replace [Region]/[City] with your actual location (e.g., America/New_York)
ln -sf /usr/share/zoneinfo/[Region]/[City] /etc/localtime && hwclock --systohc

# 2. Localization
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
echo "FONT=ter-124b" >> /etc/vconsole.conf

# 3. Network Identity
echo "thinkpad-t480" > /etc/hostname
echo "127.0.0.1 thinkpad-t480.localdomain thinkpad-t480" >> /etc/hosts
sed -i 's/#ParallelDownloads = 5/ParallelDownloads = 10/' /etc/pacman.conf

# 4. Users & Privileges
passwd
useradd -m -G wheel,video,input,storage,lp,audio -s /bin/bash username
passwd username

# 5. Sudoers
# Uncomment '%wheel ALL=(ALL:ALL) ALL'
EDITOR=vim visudo

# 6. Enable Core Services
systemctl enable NetworkManager bluetooth firewalld tlp throttled

```

### 4. Initramfs & Single-Password Unlock

Generate a secure keyfile and embed it into the boot image. This allows the Kernel to unlock the drive automatically using credentials passed from GRUB, eliminating the need to type your password a second time.

```bash
# 1. Create a 512-byte keyfile (The "Secret Handshake")
dd if=/dev/urandom of=/crypto_keyfile.bin bs=512 count=1
chmod 600 /crypto_keyfile.bin

# 2. Add Key to LUKS (Type your encryption password when asked)
cryptsetup luksAddKey /dev/nvme0n1p2 /crypto_keyfile.bin

```

**Edit `/etc/mkinitcpio.conf**`:
Modify the `FILES`, `MODULES`, and `HOOKS` lines to match exactly below.

```bash
# 1. Embed the keyfile
FILES=(/crypto_keyfile.bin)

# 2. Early Graphics Loading
MODULES=(btrfs i915 nouveau)

# 3. Hooks (Order is Critical)
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt resume filesystems fsck)

```

**Generate the image:**

```bash
mkinitcpio -P

```

---

## Phase 4: Bootloader & Disaster Recovery <a name="phase-4"></a>

You'll need the physical offset of the swapfile so the Kernel knows where to find the hibernation image on the encrypted disk.
### 1. Calculate Resume Offset

```bash
# 1. Run this command to get your specific offset
# Note: Your value is likely around 533760, but it varies by install.
btrfs inspect-internal map-swapfile -r /swap/swapfile

# REMEMBER / COPY DOWN THE OUTPUT NUMBER. This guide refers to this as [OFFSET].
```

### 2. Configure GRUB

Configure GRUB to unlock the drive (Slot 0) and pass the `crypto_keyfile.bin` to the Kernel (Slot 1) so you only type your password once.

**Edit `/etc/default/grub**`:

```bash
# 1. Enable Cryptodisk (Critical for LUKS2 boot)
GRUB_ENABLE_CRYPTODISK=y

# 2. Kernel Parameters
# Replace [UUID_P2] with the UUID of your LUKS partition (/dev/nvme0n1p2)
# Replace [OFFSET] with the number you found in Step 1
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=[UUID_P2]:cryptroot cryptkey=rootfs:/crypto_keyfile.bin root=/dev/mapper/cryptroot rootflags=subvol=@ resume=/dev/mapper/cryptroot resume_offset=[OFFSET] i915.enable_psr=0 nowatchdog"

```

**Install Bootloader:**

```bash
# Install to the EFI partition
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB

# Generate the config
grub-mkconfig -o /boot/grub/grub.cfg

```

### 3. Snapper Configuration

Initialize the snapshot structure and configure cleanup rules to prevent the disk from filling up with old backups.

```bash

# 1. Initialize Root Config
umount /.snapshots
rm -r /.snapshots
snapper --no-dbus -c root create-config /
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -a
chmod 750 /.snapshots

# 2. Configure Cleanup Rules
# Use moderate limits (50 total snapshots) to balance history vs disk usage.
snapper --no-dbus -c root set-config "TIMELINE_CREATE=yes"
snapper --no-dbus -c root set-config "NUMBER_LIMIT=50"
snapper --no-dbus -c root set-config "NUMBER_LIMIT_IMPORTANT=10"
snapper --no-dbus -c root set-config "TIMELINE_LIMIT_HOURLY=10"
snapper --no-dbus -c root set-config "TIMELINE_LIMIT_DAILY=14"
snapper --no-dbus -c root set-config "TIMELINE_LIMIT_WEEKLY=0"
snapper --no-dbus -c root set-config "TIMELINE_LIMIT_MONTHLY=0"
snapper --no-dbus -c root set-config "TIMELINE_LIMIT_YEARLY=0"

# Enable the background timer to enforce the limits set above
systemctl enable snapper-cleanup.timer
```

### 4. Snapper-Rollback (The "Ambit" Fix)

This tool allows you to rollback the system from a read-only snapshot to a read-write state without complex command-line btrfs surgery.

```bash
# 1. Switch to your user to install AUR helper (yay)
su - username
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay

# 2. Install Snapper-Rollback
yay -S snapper-rollback

# 3. Configure the Tool
# Warning: The syntax below is specific to the Python version.
exit # Switch back to root
```

**Create / Edit: `/etc/snapper-rollback.conf`:**


```ini
[root]
# 1. The Linux Root (Must be 'subvol_main' per the docs)
subvol_main = @

# 2. The Snapshots Subvolume
subvol_snapshots = @snapshots

# 3. Where to perform the swap (Standard temp folder)
mountpoint = /mnt

# 4. Your LUKS Drive (This allows the script to auto-mount it)
dev = /dev/mapper/cryptroot
```

**Enable Automatic Boot Snapshots:**

```bash
# This creates a snapshot every time you install packages
systemctl enable grub-btrfsd
```

---

## Phase 5: Desktop & Connectivity Stack <a name="phase-5"></a>

**Objective:** Install the **Hybrid Desktop Environment** (KDE Plasma + Hyprland) and the Color-Correct Printing stack.

### 1. The "Hybrid" Desktop Core

Install both environments. **SDDM** acts as the gatekeeper, allowing you to choose between "Hyprland" and "Plasma" at login.

```bash
# 1. Display Manager (Login Screen) & Xorg Backend
pacman -S sddm sddm-kcm xf86-video-nouveau

# 2. Hyprland Suite (The Tiling Compositor)
pacman -S hyprland hyprpaper hyprlock hypridle waybar rofi swaync \
    xdg-desktop-portal-hyprland qt5-wayland qt6-wayland \
    grim slurp wl-clipboard nwg-look wlogout gum xdg-desktop-portal-gtk

# 3. KDE Plasma Suite (The Full Desktop)
# Minimal install + specific tools to avoid bloat
pacman -S plasma-desktop plasma-nm plasma-pa kscreen kinfocenter \
    dolphin konsole ark kate spectacle gwenview bluedevil okular \
    xdg-desktop-portal-kde kde-gtk-config breeze-gtk plasma-firewall \
    ffmpegthumbs kdegraphics-thumbnailers kwallet-manager kwallet-pam

# 4. Theming & Qt/GTK Integration
pacman -S kvantum papirus-icon-theme qt6ct
```

### 2. User Applications

The specific suite of tools currently installed on the system.

```bash
# 1. Core Applications & Network Filesystems
# gvfs-smb/nfs/sshfs are critical for browsing NAS/Windows shares in Thunar/Dolphin
pacman -S firefox kitty thunar thunar-archive-plugin obsidian \
	vlc vlc-plugins-all libreoffice-fresh gvfs gvfs-smb gvfs-nfs \
	sshfs

# 2. Network & Bluetooth GUI Tools
# (Backends were installed in Phase 3; these are the UI controls)
pacman -S network-manager-applet blueman

# 3. Audio + Brightness Controls
pacman -S pavucontrol pipewire-alsa pipewire-pulse brightnessctl libldac

# 4. CLI Utilities 
pacman -S btop fastfetch s-tui stress

```

### 3. Printing & Scanning Stack (IPP + Color Management)

Use **IPP Everywhere** (Driverless) for printing but install proprietary drivers for scanning and `colord` for accurate color profiles. This example uses an epson printer / scanner combo.
 
```bash
# 1. CUPS & System Config
pacman -S cups system-config-printer

# 2. Discovery Protocols (Required for IPP/Network Printers)
pacman -S avahi nss-mdns

# 3. Color Management (Critical for correct RGB/CMYK conversion)
pacman -S colord

```

### 4. AUR Packages (Yay)

Use the AUR for proprietary scanner drivers and specific browser binaries.

```bash
# 1. Switch to user
su - username

# 2. Install AUR Packages
# brave-bin: Brave Browser
# epsonscan2: Proprietary Scanner Utility (Critical for ET-2760 scanning)
# epson-inkjet-printer-escpr: Drivers installed as fallback/dependency
yay -S brave-bin epsonscan2 epson-inkjet-printer-escpr epson-inkjet-printer-escpr2

# 3. Return to root
exit

```

### 5. Enable User-Facing Services


```bash
# 1. Enable Login Screen
systemctl enable sddm

# 2. Enable Printing & Discovery
# Note: Avahi (mDNS) is required for IPP Everywhere to find the printer.
systemctl enable cups avahi-daemon

```

### 6. Basic Hyprland Configuration

Instead of writing a config from scratch, copy the default template provided by the package and fix the menu launcher.

```bash
# 1. Switch to User (CRITICAL) 
su - username

# 2. Create directory (if user)
mkdir -p ~/.config/hypr

# 3. Copy the default configuration
cp /usr/share/hypr/hyprland.conf ~/.config/hypr/hyprland.conf

# 4. Patch the config to use 'rofi' instead of 'wofi'
sed -i 's/wofi --show drun/rofi -show drun/g' ~/.config/hypr/hyprland.conf

# 5. (Optional) Disable the autogenerated warning
sed -i 's/autogenerated = 1/autogenerated = 0/g' ~/.config/hypr/hyprland.conf

exit # Switch back to root
```

### 7. Virtualization Stack (KVM/QEMU)

```bash
# 1. Install Virtualization Suite
# This creates the /var/lib/libvirt directory structure
pacman -S qemu-desktop libvirt virt-manager dnsmasq dmidecode

# 2. Critical Performance Step: Nested Subvolume
# Create a separate subvolume for VM images to prevent Snapper from 
# snapshotting them (which causes severe fragmentation and performance loss).
rm -rf /var/lib/libvirt/images  # Remove the default empty folder
btrfs subvolume create /var/lib/libvirt/images
chattr +C /var/lib/libvirt/images

# 3. Permissions & Groups
# Add user to libvirt group
usermod -aG libvirt username

# Grant user access to the images folder (Fixes permission errors in virt-manager)
setfacl -m u:username:rwx /var/lib/libvirt/images

# 4. Enable Libvirt Daemon
systemctl enable libvirtd

```

**Performance Note:** When configuring VMs in `virt-manager`, manually edit the XML or select "Customize configuration before install."

Under the **Disk** section, ensure the following settings to fully utilize the NoCoW setup:
* ***Storage format:** `raw` (Max Performance) or `qcow2` (VM Snapshots enabled)
- **Cache mode:** None
- **I/O mode:** Native

**XML Snippet:**

```xml
<driver name='qemu' type='qcow2' cache='none' io='native'/>
```

### 8. Firewall Security Profiles

```bash
# Note: Use 'firewall-offline-cmd' because the daemon is not running in chroot.

# 1. Set Default Zone to 'Public' (Restrictive)
firewall-offline-cmd --set-default-zone=public

# 2. Configure 'Home' Zone (Trusted)
# Allow mDNS (Printer Discovery) and Samba (File Sharing)
firewall-offline-cmd --zone=home --add-service=mdns
firewall-offline-cmd --zone=home --add-service=samba-client

```

---

## Phase 6: Power, Hardware & Resiliency <a name="phase-6"></a>

**Objective:** Implement the "0W Nvidia" power policy, apply CPU undervolting, and fix T480 hardware quirks.

### 1. The Nvidia Power Disable (Via TLP)

Instead of blacklisting drivers (which can break external HDMI), use TLP's Runtime Power Management to force the Nvidia card and its controller into deep sleep.

**Edit `/etc/tlp.conf`:**

Find these specific lines and set them exactly as shown or add them if necessary. This configuration disables WiFi power saving (for stability) and forces the Nvidia card (`01:00.0`) to sleep.

```ini
# 1. WiFi Stability (Prevents random drops on AX210)
WIFI_PWR_ON_AC=off
WIFI_PWR_ON_BAT=off

# 2. Disable USB Autosuspend (Prevents mouse/keyboard lag)
USB_AUTOSUSPEND=0

# 3. Battery Charge Thresholds (Prolongs battery lifespan)
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
START_CHARGE_THRESH_BAT1=75
STOP_CHARGE_THRESH_BAT1=80

# 4. The 0W Nvidia Fix
# 01:00.0 = Nvidia MX150
# 00:1c.0 = PCIe Root Port for Nvidia
RUNTIME_PM_ENABLE="01:00.0 00:1c.0"
```

### 2. Lock-Down Undervolts (`throttled`)

This configuration applies a -100mV undervolt to the CPU core and Cache, significantly reducing heat and throttling.

**Write `/etc/throttled.conf`:**

```ini, TOML
[GENERAL]
Enabled: True
Sysfs_Power_Path: /sys/class/power_supply/AC*/online
Autoreload: True

[BATTERY]
Update_Rate_s: 5
PL1_Tdp_W: 29
PL1_Duration_s: 28
PL2_Tdp_W: 44
PL2_Duration_S: 0.002
Trip_Temp_C: 85
cTDP: 0
Disable_BDPROCHOT: False

[AC]
Update_Rate_s: 5
PL1_Tdp_W: 44
PL1_Duration_s: 28
PL2_Tdp_W: 44
PL2_Duration_S: 0.002
Trip_Temp_C: 95
cTDP: 0
Disable_BDPROCHOT: False

[UNDERVOLT.BATTERY]
CORE: -100
GPU: -50
CACHE: -100
UNCORE: -80
ANALOGIO: 0

[UNDERVOLT.AC]
CORE: -100
GPU: -50
CACHE: -100
UNCORE: -80
ANALOGIO: 0

```

**Enable the Service:**


```bash
systemctl enable throttled

```

**Note**: If you find the above end up getting overwritten and you've ensured the system is stable you can lock the settings via the following:

```bash
# Optional: Only due this after you've confirmed the system stability
chattr +i /etc/throttled.conf

```

### 3. T480 Hardware Stability Fixes (`modprobe.d`)

Create specific kernel module options to fix known T480 freezes and glitches.

```bash
# 1. Fix SD Card Reader Freezes (Critical)
# The O2 Micro reader is known to hang the system without this quirk.
echo "options sdhci_pci debug_quirks2=0x4" > /etc/modprobe.d/sdhci.conf

# 2. WiFi Stability (Redundant safety for AX210)
echo "options iwlwifi power_save=0" > /etc/modprobe.d/iwlwifi.conf

# 3. Watchdog Fix (Prevents hang on reboot)
echo "blacklist iTCO_wdt" > /etc/modprobe.d/watchdog-fix.conf

```

---

## Phase 7: Validation & Maintenance <a name="phase-7"></a>

**Objective:** Final reboot and verification of all systems.

### 1. Finalize Installation


```bash
exit
umount -R /mnt
reboot

```

### 2. Post-Reboot Validation

After logging in (Select "Hyprland" or "Plasma" at the SDDM screen), verify the critical systems:

**A. Verify 0W Nvidia Power Draw:**


```bash
# The state should be "D3cold" (Powered off)
cat /sys/bus/pci/devices/0000:01:00.0/power_state

```

**B. Verify Undervolt Application:**


```bash
sudo throttled --monitor
# Look for "CORE: -100.0 mV" in the output

```

**C. Verify Snapper Rollback Capability:**

```bash
# List your current snapshots
snapper list

# Verify the rollback tool config is valid
cat /etc/snapper-rollback.conf

```
---

## Acknowledgments

This guide was created through extensive trial and error on my physical hardware while referencing the Arch and Hyprland Wikis. 

However, I also want to acknowledge the role of AI as a research partner—specifically for troubleshooting obscure errors, structuring the documentation, and the occasional ridiculous argument when the machine was wrong. As always: trust, but verify.

---

## License

Distributed under the MIT License. See `LICENSE` for more information.

---

*Last Updated: February 2026 | Kernel: Linux LTS (6.x) | Config Status: Immutable*
