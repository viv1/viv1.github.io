---
layout: post
title: Raspberry Pi 4 - Booting from USB SSD instead of SD Card
date: 2026-03-29 15:00
category: Guides
tags: ["raspberry-pi", "ssd", "usb-boot", "storage", "sd-card"]
description: A practical reference for moving Raspberry Pi 4 from SD card boot to USB SSD boot — commands, expected output, and common pitfalls.
---

- [Why Move to SSD?](#why-move-to-ssd)
- [Prerequisites](#prerequisites)
- [Step 1: Check Your Bootloader](#step-1-check-your-bootloader)
- [Step 2: Check Boot Order](#step-2-check-boot-order)
- [Step 3: Partition the SSD](#step-3-partition-the-ssd)
- [Step 4: Copy the Boot Partition](#step-4-copy-the-boot-partition)
- [Step 5: Copy the Root Filesystem](#step-5-copy-the-root-filesystem)
- [Step 6: Fix fstab on the SSD](#step-6-fix-fstab-on-the-ssd)
- [Step 7: Boot from SSD](#step-7-boot-from-ssd)
- [Keeping the SD Card as Fallback](#keeping-the-sd-card-as-fallback)
- [Common Pitfalls](#common-pitfalls)

## Why Move to SSD?

SD cards are slow and wear out. A USB SSD gives you better performance, more storage, and longer lifespan. The Pi 4 supports USB boot natively through its EEPROM bootloader. I recently (re)decided to move back to SSD again. Chalking down how I did it.

## Prerequisites

You need:
- Raspberry Pi 4 with a working SD card setup
- USB SSD (with a USB 3.0 enclosure or USB-to-SATA adapter)
- SSH access to your Pi

## Step 1: Check Your Bootloader

The Pi 4 has a dedicated EEPROM chip that runs before the OS loads — it's the first piece of code that executes and decides where to look for a bootable partition. USB boot support was added to this EEPROM in September 2020, so older Pi 4 units need an update.

SSH into your Pi and verify the EEPROM supports USB boot:

```sh
vcgencmd bootloader_version
```

```
2023/01/11 17:40:52
version 8ba17717fbcedd4c3b6d4bce7e50c7af4155cba9 (release)
timestamp 1673458852
```

If your version is older than **September 2020**, update it:

```sh
sudo rpi-eeprom-update -a
sudo reboot
```

## Step 2: Check Boot Order

The EEPROM has a `BOOT_ORDER` setting that controls which devices the Pi tries to boot from, and in what order. Each hex digit represents a boot device.

```sh
vcgencmd bootloader_config | grep BOOT_ORDER
```

```
BOOT_ORDER=0xf14
```

Read the hex right-to-left: `4` = USB, `1` = SD card, `f` = restart loop. This means: try USB first, fall back to SD.

If USB boot isn't enabled, set it:

```sh
sudo raspi-config
# Advanced Options → Boot Order → USB Boot
```

## Step 3: Partition the SSD

The Pi's boot process requires two separate partitions:

- **FAT32 boot partition** (~256MB): The Pi's GPU firmware can only read FAT filesystems. It loads `bootcode.bin`, `start4.elf`, the kernel, and device tree files from here. This is a hardware limitation — not a choice.
- **ext4 root partition** (rest of the disk): This is where the actual Linux OS lives — `/usr`, `/etc`, `/home`, etc. ext4 is the standard Linux filesystem with journaling, permissions, and symlinks that Linux needs to operate.

We also use **MBR** (msdos) partitioning instead of GPT, because the Pi 4's bootloader expects an MBR partition table.

Plug the SSD into the Pi. Check it's detected:

```sh
lsblk
```

```
sda           8:0    0 223.6G  0 disk
mmcblk0     179:0    0  14.9G  0 disk
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0  14.6G  0 part /
```

Partition the SSD using `fdisk`:

```sh
sudo fdisk /dev/sda
```

Inside `fdisk`, run these commands in order:

1. `o` — create a new MBR partition table
2. `n` → `p` → `1` → press Enter for default start → `+256M` — creates the 256MB boot partition
3. `t` → `c` — sets the partition type to W95 FAT32 (LBA)
4. `n` → `p` → `2` → press Enter → press Enter — creates the root partition using all remaining space
5. `w` — writes the changes and exits

Verify the layout:

```sh
lsblk /dev/sda
```

```
sda      8:0    0 223.6G  0 disk
├─sda1   8:1    0   256M  0 part
└─sda2   8:2    0 223.3G  0 part
```

Now format both partitions. Creating a partition and formatting it are two separate steps — the partition table only defines boundaries, the filesystem makes it usable:

```sh
sudo mkfs.vfat -F 32 -n BOOT /dev/sda1
sudo mkfs.ext4 -L root /dev/sda2
```

## Step 4: Copy the Boot Partition

The boot partition contains the Pi's firmware and kernel. The key files are:

- `bootcode.bin` — first-stage bootloader (loaded by the GPU)
- `start4.elf` / `fixup4.dat` — GPU firmware for Pi 4
- `kernel*.img` — the Linux kernel
- `*.dtb` files — device tree blobs that tell the kernel about the Pi's hardware
- `config.txt` — Pi's equivalent of a BIOS settings screen
- `cmdline.txt` — kernel boot parameters, including which partition is the root filesystem

Always copy these from your **working SD card**, not from a different OS image. The firmware version must match the OS on your root partition — mixing Bookworm boot files with a Buster root filesystem will fail.

```sh
sudo mount /dev/sda1 /mnt
sudo cp -r /boot/* /mnt/
```

Update `cmdline.txt` on the SSD to point to the SSD's root partition:

```sh
sudo sed -i 's|root=/dev/mmcblk0p[0-9]*|root=/dev/sda2|' /mnt/cmdline.txt
cat /mnt/cmdline.txt
```

You should see `root=/dev/sda2` in the output.

```sh
sudo umount /mnt
```

## Step 5: Copy the Root Filesystem

We use `rsync -ax` to clone the root filesystem. The flags matter:

- `-a` (archive) preserves permissions, ownership, symlinks, and timestamps — all essential for a working Linux system
- `-x` stays on one filesystem, so it won't follow mounts into `/boot` or other partitions

We exclude virtual filesystems (`/proc`, `/sys`, `/dev`, `/run`) because these are populated by the kernel at runtime — they don't contain real files. `/tmp` is excluded because it's ephemeral. `/boot` is excluded because the SSD has its own boot partition.

```sh
sudo mount /dev/sda2 /mnt
sudo rsync -ax / /mnt/ --exclude=/boot --exclude=/media --exclude=/mnt --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run --exclude=/tmp
```

After the copy, we recreate the empty mount point directories. The kernel needs these directories to exist so it can mount the virtual filesystems on boot:

```sh
sudo mkdir -p /mnt/{boot,media,mnt,proc,sys,dev,run,tmp}
sudo chmod 1777 /mnt/tmp
```

## Step 6: Fix fstab on the SSD

`/etc/fstab` tells Linux which partitions to mount and where, every time the system boots. Since we cloned from the SD card, the SSD's fstab still references `mmcblk0` (the SD card device name). We need to update it to `sda` (the USB SSD device name), otherwise the system won't mount partitions correctly after boot.

```sh
cat /mnt/etc/fstab
```

```
/dev/mmcblk0p1  /boot  vfat  defaults  0  2
/dev/mmcblk0p2  /      ext4  defaults,noatime  0  1
```

Update to reference the SSD:

```sh
sudo sed -i 's|/dev/mmcblk0p[0-9]*|/dev/sda1|' /mnt/etc/fstab
sudo sed -i 's|sda1.*ext4|sda2       /               ext4|' /mnt/etc/fstab
cat /mnt/etc/fstab
```

Should now show:

```
/dev/sda1  /boot  vfat  defaults  0  2
/dev/sda2  /      ext4  defaults,noatime  0  1
```

Unmount:

```sh
sudo umount /mnt
```

## Step 7: Boot from SSD

Shut down the Pi. USB devices take longer to initialize than SD cards, so the Pi may take a few extra seconds to boot compared to what you're used to.

```sh
sudo shutdown -h now
```

Remove the SD card, power on. Give it 30-60 seconds, then SSH in and verify:

```sh
df -h /
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       219G   12G  197G   6% /
```

## Keeping the SD Card as Fallback

You can leave both plugged in. With `BOOT_ORDER=0xf14`:

| SSD plugged in? | SD card in? | Boots from |
|---|---|---|
| Yes | Yes | SSD |
| Yes | No | SSD |
| No | Yes | SD card |

Each has its own `cmdline.txt` pointing to its own root, so there's no conflict.

## Common Pitfalls

**"VFS: Unable to mount root fs on unknown-block"**
- This means the kernel loaded fine from the boot partition, but couldn't find or mount the root filesystem. Either the root partition isn't formatted, or `cmdline.txt` has the wrong `root=` value
- Verify with `lsblk -f` — if FSTYPE is blank, the partition has no filesystem. A partition can exist in the partition table without having an actual filesystem on it — creating a partition and formatting it are two separate steps

**Monitor goes to standby / no display**
- HDMI settings in `config.txt` might not match your monitor
- Try adding `hdmi_force_hotplug=1` to `config.txt`

**Firmware version mismatch**
- The boot files must match your OS version. Don't use Bookworm boot files with a Buster root filesystem
- Always copy boot files from your working SD card, not from a different OS image

**Formatting from macOS**
- `diskutil eraseVolume` struggles with MBR partitions
- Use `newfs_msdos -F 32 -v BOOT /dev/rdiskXsN` (note the `r` prefix for raw disk)
- Always `diskutil unmountDisk` first

**Forgot to update fstab**
- System boots but partition mounts break
- Always check `/etc/fstab` on the SSD root after cloning
