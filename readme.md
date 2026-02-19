# Arch Linux + Windows Dual Boot Installation Guide 🦅🪟

A straightforward, step-by-step manual for manually installing Arch Linux alongside an existing Windows installation on a UEFI system. 

> **Disclaimer:** Partitioning disks and modifying bootloaders carries the risk of data loss. **Always back up your important files in Windows before starting this process.**

---

## ⚠️ Pre-requisites

Before booting the Arch Linux live USB, you must prepare your Windows system and BIOS/UEFI:

1. **Create Unallocated Space:** Boot into Windows, open **Disk Management**, right-click your main partition (usually `C:`), and select **Shrink Volume**. Leave the shrunk space as "Unallocated" (do *not* format it). 
2. **Disable Fast Startup:** In Windows Control Panel -> Power Options -> "Choose what the power buttons do", uncheck **Turn on fast startup**.
3. **Disable Secure Boot:** Restart into your BIOS/UEFI settings and disable Secure Boot. Ensure your SATA mode is set to **AHCI** (not RAID/RST).
4. **Create Bootable USB:** Flash the [latest Arch ISO](https://archlinux.org/download/) to a USB drive using Rufus or BalenaEtcher.

---

## 1. Pre-Installation

Boot into the Arch live environment.

**Verify UEFI Mode:**
```bash
ls /sys/firmware/efi/efivars

```

*(If the directory is full of files, you are successfully booted in UEFI mode.)*

**Connect to the Internet:**
If you are using Ethernet, you should be connected automatically. For Wi-Fi:

```bash
iwctl
# Inside the iwctl prompt:
# station wlan0 scan
# station wlan0 get-networks
# station wlan0 connect "YourNetworkName"
# exit

```

**Verify Connection & Update Clock:**

```bash
ping -c 3 archlinux.org
timedatectl set-ntp true

```

---

## 2. Partitioning

We will use `cfdisk` for an interactive and visual partitioning experience.

```bash
lsblk        # Identify your target drive (e.g., /dev/nvme0n1 or /dev/sda)
cfdisk /dev/nvme0n1

```

Locate the **Free Space** (the unallocated space you made in Windows) and create the following partitions:

* **Root (`/`):** Allocate the majority of your free space. Change the type to **Linux filesystem**.
* **Swap (Optional):** Allocate 4GB - 8GB. Change the type to **Linux swap**.

> **Note:** Do **NOT** create a new EFI system partition. We will use the existing Windows EFI partition (usually a 100MB-300MB FAT32 partition) to host the GRUB bootloader.

Write the changes and type `yes` to confirm.

---

## 3. Formatting & Mounting

Double-check your partition names using `lsblk` before running these commands. In this example, `p1` is the Windows EFI, `p6` is Linux Root, and `p7` is Swap.

**Format the partitions:**

```bash
# DO NOT format the Windows EFI partition!
mkfs.ext4 /dev/nvme0n1p6  # Format Root
mkswap /dev/nvme0n1p7     # Format Swap

```

**Mount the partitions:**

```bash
mount /dev/nvme0n1p6 /mnt               # Mount Root
mount --mkdir /dev/nvme0n1p1 /mnt/boot  # Mount existing Windows EFI to /boot
swapon /dev/nvme0n1p7                   # Enable Swap

```

---

## 4. Base Installation

Install the essential base system packages, Linux kernel, firmware, and text editor.

```bash
pacstrap -K /mnt base linux linux-firmware nano git

```

Generate the filesystem table so the system knows how to mount partitions on boot:

```bash
genfstab -U /mnt >> /mnt/etc/fstab

```

Change root into your new system:

```bash
arch-chroot /mnt

```

---

## 5. System Configuration

**Timezone:**

```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc

```

**Localization:**
Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` (and any other needed locales).

```bash
nano /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

```

**Network & Hostname:**

```bash
echo "myarch" > /etc/hostname
pacman -S networkmanager
systemctl enable NetworkManager

```

**Set Root Password:**

```bash
passwd

```

---

## 6. User Setup

It's dangerous to run as root all the time. Create a standard user and give them `sudo` privileges.

```bash
pacman -S sudo
useradd -mG wheel username
passwd username

```

Uncomment `%wheel ALL=(ALL:ALL) ALL` in the sudoers file:

```bash
EDITOR=nano visudo

```

---

## 7. Bootloader (GRUB)

To detect Windows alongside Arch, you must install GRUB, EFI tools, and `os-prober`.

```bash
pacman -S grub efibootmgr os-prober

```

**Enable OS Prober:**
Edit the GRUB configuration file to allow it to scan for Windows:

```bash
nano /etc/default/grub
# Uncomment or add this line:
# GRUB_DISABLE_OS_PROBER=false

```

**Install and Generate GRUB:**

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

```

*(You should see Windows Boot Manager listed in the output of the mkconfig command.)*

---

## 8. Finalize and Reboot

You're done! Exit the chroot environment, unmount your drives, and restart.

```bash
exit
umount -R /mnt
reboot

```

Remove your USB drive. You should now be greeted by the GRUB boot menu, offering both Arch Linux and Windows! 🎉

```

***

Would you like me to add a specific section to this README, such as how to install a Desktop Environment (like GNOME, KDE Plasma, or Hyprland) or setup specific graphics drivers after the first reboot?

```