# Emulating ARM Cortex-A72 (Raspberry Pi 3 Model B+)
This repository provides a reproducible setup for emulating a 64bit ARM-Cortex A72 system using QEMU and KVM.
However, the performance doesn't match that of the real hardware, not suitable to run benchmark or to be used in production.

> Based on the excellent guide by [xeome.dev](https://notes.xeome.dev/notes/Emulating-Cortex-A72) with modifications and enhancements for personal use.

## Requirements
- Ubuntu 22.04+ (host)
- QEMU with KVM support for `aarch64`
- A bit of your patient
- Other tools:
  - `wget` to download the OS image or use your browser
  - `xz` to decompress the image
  - `parted` to resize the image

## Install dependencies:

For setting up QEMU/KVM, you can follow the excellent guide by [sysguides.com](https://sysguides.com/install-kvm-on-linux). Follow step 1, 2, 3, 5, 6.
Install QEMU for `aarch64`:
- Ubuntu: `sudo apt install qemu-system-arm`
- Arch: `sudo pacman -S qemu-system-aarch64` or `yay -S qemu-system-aarch64`

## Create a project directory

Create a directory to store the project files:
```bash
mkdir rpi_image
cd rpi_image
```

## Download and decompress the Raspberry Pi OS Lite

```bash
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-11-19/2024-11-19-raspios-bookworm-arm64-lite.img.xz
xz -d -k 2024-11-19-raspios-bookworm-arm64-lite.img.xz
```
## Extract the kernel and device tree blob

If you plan to use the same image as I, you can use the provided `kernel8.img` and `bcm2710-rpi-3-b-plus.dtb`. Go to [Resize the image](#resize-the-image)

### Mount partitions inside the image

Use `losetup` to create loop devices:
```bash
sudo losetup --show -Pf 2024-11-19-raspios-bookworm-arm64-lite.img
# Output
/dev/loop25
```
- `loop25p1` is your `boot/` inside the image
- `loop25p2` is your `root/` image base directory 

Mount them:
```bash
sudo mkdir /mnt/raspi
sudo mount /dev/mapper/loop25p2 /mnt/raspi
sudo mount /dev/mapper/loop25p1 /mnt/raspi/boot
```

### Copy the kernel and device tree blob

Copy the two files to the `rpi_image` that you created above
```bash
cp /mnt/raspi/boot/kernel8.img ./
cp /mnt/raspi/boot/bcm2710-rpi-3-b-plus.dtb ./
```

### Unmount the image

**This part is important**, otherwise your image might be corrupted when you start QEMU.
```bash
sudo umount /mnt/raspi/boot
sudo umount /mnt/raspi
sudo losetup -d /dev/loop25
```

## Resize the image

The image partition size might be too small if we want to do anything with it, we can resize the image and expand the partition to fix this.

```bash
qemu-img resize 2024-11-19-raspios-bookworm-arm64-lite.img 32G
losetup --show -Pf raspios-bookworm-arm64-lite.img  # Just create a new loop device, do not mount
```

We use `gparted` to resize the partition
```bash
sudo parted /dev/loop25
(parted) resizepart 2 100%  # Expand the second partition to the entirety of the the image
```

We grow the `ext4` filesystem
```bash
sudo e2fsck -f /dev/loop25p2
sudo resize2fs /dev/loop25p2
```

Sanity check to make sure we got it
```bash
(parted) print
Model: Loopback device (loopback)
Disk /dev/loop25: 34.4GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  541MB   537MB   primary  fat32        lba
 2      541MB   32.2GB  31.7GB  primary  ext4
```

Detach the loop device
```bash
sudo losetup -d /dev/loop25
```

## Start QEMU
Here I chose Cortex-a72 as the CPU to closely match Raspberry Pi 4, as this board is not entirely emulated.

```bash
qemu-system-aarch64 \
    -machine raspi3b \
    -cpu cortex-a72 \
    -dtb ./bcm2710-rpi-3-b-plus.dtb \
    -m 1G \
    -smp 4 \
    -kernel ./kernel8.img \
    -sd ./2024-11-19-raspios-bookworm-arm64-lite.img  \
    -append "rw earlyprintk loglevel=8 console=ttyAMA0,115200 console=tty highres=off console=ttyAMA0 dwc_otg.lpm_enable=0 root=/dev/mmcblk0p2 rootdelay=1 modules-load=dwc2,g_ether" \
    -device usb-net,netdev=eth0 \
    -netdev user,id=eth0,hostfwd=tcp::2222-:22 \
    -device usb-kbd
```

## Problem encountered

During the guest OS initialization, you might face these errors. They can be ignored since there is still outbound internet connection. However, SSH connection to the machine doesn't work.

```bash
usbnet: failed control transaction: request 0x8006 value 0x600 index 0x0 length 0xa
usbnet: failed control transaction: request 0x8006 value 0x600 index 0x0 length 0xa
usbnet: failed control transaction: request 0x8006 value 0x600 index 0x0 length 0xa
```

If you can make it work, you can ssh into the emulator by:

```bash
ssh username@localhose -p 2222
```
