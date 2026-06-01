# Ubuntu + Btrfs + Automatic Snapshots

This is a fork of the https://github.com/diogopessoa/ubuntu-btrfs-install repo to address some issues I have found with the original script when used on a Ubuntu Server 26.04 virtual machine and make some changes that I find useful. Namely, my fork seeks to carry out the following changes:

- Align the Btrfs subvolumes with that used by CachyOS - As such, the script no longer creates the @libvirt, @flatpak, @docker, @containers, @machines and @opt subvolumes and instead creates the @root and @srv subvolumes.
- Fix the root:root ownership of the initial user's home folder - By moving the contents of the /home folder to the corresponding @home subvolume, this preserves the 1000:1000 UID and GID of the initial user's home folder that causes Nano, Fish, etc. to error out when logging into the new installation for the first time.
- Fix the loss of data in the /var/log and /var/cache folders - By fixing the path to the folders to point to ./@/var/cache and ./@/var/log in the script.
- Fix the loss of a dedicated swap partition - By commenting out the call to SED to delete the Swap partition entry from the /etc/fstab file allowing the system to use the Swap partition if you created one in the installer.
- Enable Level 3 zstd compression on all Btrfs subvolumes - This further aligns Ubuntu with the CachyOS Btrfs install by using a higher compression algorithm for data on the subvolume.
- Fix the error from systemd about duplicate mount points for /boot and /boot/efi - By adding a call to SED to delete the existing /boot and /boot/efi entries from /etc/fstab, we can avoid an error from systemd about failing to mount the volumes due to them being duplicated in /etc/fstab. As a result of this, the /boot/efi permissions are correctly set to only allow Root to access the volume.

My test environment for this fork is a Ubuntu Server 26.04 virtual machine on VMware ESXi with UEFI, secureboot enabled and a 64 GB virtual hard disk which mapped to /dev/sda. However, I don't believe there is any reason why this fork should not work on earlier versions. Other than that, the changes are mainly to this readme to reflect my experiments using 26.04.

## What the Script Does

This script creates Btrfs subvolumes (while still in Live CD/USB mode) for Ubuntu 24.04 (or newer) and compatible derivatives.

- Creates Btrfs subvolumes:
  - `@home`
  - `@log`
  - `@cache`
  - `@tmp`
  - `@root`
  - `@srv`

## Requirements

- Ubuntu 24.04 or newer installed with:
  - Root filesystem using **Btrfs**
  - Separate **/boot** partition formatted as ext4 (2GB)
  - (Optional) EFI partition for UEFI systems (1GB)
  - (Optional) Swap partition
- Run the `ubuntu-btrfs-install` script from the **Live CD/USB** after Ubuntu is installed

## Install Ubuntu with Btrfs

This guide uses Ubuntu 26.04 as an example.

### Step-by-Step Guide

1. **Preparation**
   - Create a bootable USB drive using the Ubuntu ISO
   - (Optional) Disable Secure Boot in UEFI if needed to avoid installation issues
  
2. **Start Installation**
   - Boot from the USB drive and select your language
   - Choose "Manual Installation" (custom partitioning)

3. **Create Partitions in the Correct Order**
   - Create a new GPT partition table on the disk
   - Create the **/boot** partition:
     - Size: 2GB
     - Format: ext4
     - Mount Point: `/boot`
   - (Optional) Create the swap partition:
     - Size: 4GB (Or equal to RAM size if using Hibernate)
     - Format: Swap
   - Create the root **/** partition:
     - Use all remaining space
     - Format: Btrfs
     - Mount Point: `\`

Note: The `/boot/efi` partition will be created automatically when you add the `/boot` partition and set to around 1.049GB by the Ubuntu installer, you can manually select this partition and edit its size to be 1GB to align with this guide if you wish.

4. **Final Partition Table Should Look Like**
   - `/boot/efi` as FAT32 (vfat)
   - `/boot` as ext4
   - (Optional) swap
   - `/` as Btrfs
  
5. **Complete Installation**
   - Finish the Ubuntu installation and then select Help > Enter Shell **(DO NOT REBOOT YET)**
  
## How to Use the Script

⚠️ **After installing Ubuntu with Btrfs, do not reboot!**

### Identify Your Partitions

Run the following command in the terminal:

```bash
lsblk -f
```

Look for identifiers like `sda`, `nvme0n1`, etc. Example output:

```
sda     
├─sda1  vfat   /target/boot/efi
├─sda2  ext4   /target/boot
└─sda3  btrfs  /target
```

### Download the Script to the Live Session

```bash
mkdir ~/Downloads
cd ~/Downloads
wget https://raw.githubusercontent.com/sirwobbythefirst/ubuntu-btrfs-install/main/ubuntu-btrfs-install.sh
```

### Make it Executable

```bash
chmod +x ubuntu-btrfs-install.sh
```

### Run the Script

The argument order must be: `root` ▶️ `boot` ▶️ `efi`

```bash
./ubuntu-btrfs-install.sh sda3 sda2 sda1
```

This example is using `/dev/sda`

> Double-check your partition names using `lsblk -f`

### ✅ Done!

You can now reboot before installing Snapper and Btrfs Assistant for automatic snapshots.

💡 Tip: to view Btrfs subvolumes run:

```bash
sudo btrfs subvolume list /
```

## 📦 Manual Installation of Snapper and Btrfs Assistant (Post-Installation)
Snapper is a snapshot manager and Btrfs Assistant is a Snapper GUI.

After rebooting the system, install:

```bash
sudo apt update
sudo apt install -y snapper btrfs-assistant
```

Create Snapper root config:

```bash
sudo snapper -c root create-config /
```

Enable timeline and cleanup timers:

```bash
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```

**Not needed on Ubuntu 26.04 as these services are automatically enabled and started by apt during install.**

You can now launch **Btrfs Assistant** from your application menu or run:

```bash
btrfs-assistant
```

### Automatic Snapshots Configuration 

1. Now go to **"Snapper Settings"** tab 🟢 **Enable timeline snapshots**:
   - Hourly save: 10
   - Daily save: 10
   - Weekly save: 0
   - Monthly save: 3
   - Yearly save: 1
   - Number save: 10
- System unit settings:

   * 🟢 Check **"Enable cleanup enabled"**
   * 🟢 Check **"Snapper timeline enabled"**
   * ❌ Keep unmarked **"Snapper boot"**
     - With a separate /boot on ext4, enabling Boot Snapshots is not recommended because:
     - They will have no real effect.
     - They may cause confusion or failures in system restores (since /boot will not be included in Btrfs snapshots).
     
2. Click **"Apply systemd changes"**.


### ✅ Done! Your system now has snapshots automatically.


### Screenshots

- Btrfs Assistant "Snapper"
![Btrfs Assistant Snapper](https://gitlab.com/-/project/32535488/uploads/65b6004c3257d66154828259a0fed47d/image.png)

- Btrfs Assistant "Snapper Settings"
![Btrfs Assistant Snapper Settings](https://gitlab.com/-/project/32535488/uploads/429be74e9fb92088697944d23a1def1d/image.png)

## License

MIT License — [View License](https://github.com/diogopessoa/ubuntu-btrfs-install/blob/main/LICENSE) You can use, modify, and contribute!

## Credits

* [Diogo Pessoa](https://github.com/diogopessoa) - Original Script Developer
* [openSUSE Team](https://github.com/openSUSE/snapper) — Snapper
* [Antynea](https://github.com/Antynea/grub-btrfs) — grub-btrfs
* [Dan Cantrell](https://gitlab.com/btrfs-assistant/btrfs-assistant) — Btrfs Assistant
* [Ubuntu](https://ubuntu.com/download) — Operating System
