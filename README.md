# Licheepi-Nano-LinuxDistro

A custom Linux distribution built for the **Lichee Pi Nano** (Allwinner F1C100s, ARM926EJ-S).  
This repository contains the build artifacts and configuration needed to flash a working Linux image to an SD card.

---

## 📁 Build Artifacts (`1-Boot-Success/`)

| File | Description |
|------|-------------|
| `u-boot-sunxi-with-spl.bin` | U-Boot bootloader with SPL (Secondary Program Loader). Written raw to SD card at offset 8KB, outside any partition. |
| `zImage` | Compressed Linux kernel image |
| `suniv-f1c100s-licheepi-nano.dtb` | Linux Device Tree Blob — describes hardware layout to the kernel |
| `rootfs.tar` | Root filesystem archive (Buildroot-generated) |

---

## 🗂️ SD Card Layout

```
SD Card (600MB total)
┌──────────────────────────────────────┐
│  Offset 8KB: u-boot-sunxi-with-spl  │  ← Raw write, NOT in partition table
├──────────────────────────────────────┤
│  Offset 1MB — Partition 1 (FAT32)   │  ← 64MB
│    zImage                            │
│    suniv-f1c100s-licheepi-nano.dtb  │
├──────────────────────────────────────┤
│  Offset 65MB — Partition 2 (ext4)   │  ← 512MB
│    rootfs (extracted from tar)       │
└──────────────────────────────────────┘
```

## 🚀 Method 1: Flash using genimage (Recommended)

### Prerequisites
```bash
sudo apt install genimage

#clone form git
git clone https://github.com/pengutronix/genimage.git
```

### Directory structure
```
lichee_workspace/
├── genimage-licheepi-nano.cfg
├── input/
│   ├── u-boot-sunxi-with-spl.bin
│   ├── zImage
│   └── suniv-f1c100s-licheepi-nano.dtb
├── rootfs/          ← extracted rootfs.tar
├── tmp/
└── output/
```

### Steps
```bash
# 1. Prepare directories
mkdir -p input/ rootfs/ tmp/ output/

# 2. Copy input files
cp /path/to/u-boot-sunxi-with-spl.bin input/
cp /path/to/zImage input/
cp /path/to/suniv-f1c100s-licheepi-nano.dtb input/

# 3. Extract rootfs (must use fakeroot to preserve permissions)
fakeroot tar -xf /path/to/rootfs.tar -C rootfs/

# 4. Generate SD card image
fakeroot genimage \
    --config genimage-licheepi-nano.cfg \
    --rootpath rootfs/ \
    --tmppath tmp/ \
    --inputpath input/ \
    --outputpath output/

# 5. Flash to SD card (replace sdX with your device e.g. sdb)
sudo dd if=output/sdcard.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

---

## 🔧 Method 2: Manual Flash (Step by Step)

Use this method if you want to understand each step or don't have genimage available.

### Prerequisites
```bash
sudo apt install dosfstools e2fsprogs parted
```

### Steps

```bash
# Set your SD card device — VERIFY this with lsblk before proceeding!
DEVICE=/dev/sdX

# Step 1: Write U-Boot (raw, at 8KB offset — before any partition)
sudo dd if=u-boot-sunxi-with-spl.bin of=$DEVICE bs=1024 seek=8 conv=notrunc

# Step 2: Create partition table
sudo parted $DEVICE --script mklabel msdos
sudo parted $DEVICE --script mkpart primary fat32 1MiB 65MiB
sudo parted $DEVICE --script mkpart primary ext4 65MiB 577MiB
sudo parted $DEVICE --script set 1 boot on

# Step 3: Format partitions
sudo mkfs.vfat -F 32 ${DEVICE}1
sudo mkfs.ext4 ${DEVICE}2

# Step 4: Mount and copy boot files
sudo mkdir -p /mnt/boot /mnt/root
sudo mount ${DEVICE}1 /mnt/boot
sudo cp zImage /mnt/boot/
sudo cp suniv-f1c100s-licheepi-nano.dtb /mnt/boot/
sudo umount /mnt/boot

# Step 5: Extract rootfs
sudo mount ${DEVICE}2 /mnt/root
sudo tar -xf rootfs.tar -C /mnt/root
sudo umount /mnt/root

# Step 6: Sync and eject
sync
sudo eject $DEVICE

echo "Done! Insert SD card into Lichee Pi Nano and power on."
```

> ⚠️ **Warning:** Double-check `DEVICE=/dev/sdX` with `lsblk` before running `dd` or `parted`.  
> Writing to the wrong device will overwrite your host system's disk.

---

## 🔍 Boot Flow

```
Power ON
   │
   ▼
F1C100s Boot ROM
   │  reads SPL from SD offset 8KB
   ▼
SPL (in u-boot-sunxi-with-spl.bin)
   │  initializes DRAM
   ▼
U-Boot
   │  reads zImage + DTB from FAT32 partition
   ▼
Linux Kernel
   │  mounts ext4 rootfs partition
   ▼
Buildroot userspace
```

---

## 🛠️ Build Environment

| Component | Version |
|-----------|---------|
| Buildroot | 2017.08 |
| Toolchain | GCC Linaro 7.2.1 (arm-linux-gnueabi) |
| Target SoC | Allwinner F1C100s |
| Target Board | Lichee Pi Nano |

---
