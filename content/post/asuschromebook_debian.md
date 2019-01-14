+++
math = true
date = "2018-12-04T10:00:00+02:00"
title = "Debian on Asus Chromebook Flip C101PA"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

* The Asus Chromebook Flip C101PA (bob) is a convertible touchscreen laptop powered by an ARMv8 Rockchip RK3399 hexa-core processor and 4GB RAM, measuring 10.4" x 7.2" x 0.6" and weighing 1 kg. *

Features:
* Rockchip RK3399 (OP1) dual-core 2.0GHz Cortex-A72 and quad-core 1.4GHz Cortex-A53 processor
* 4GB LPDDR3 RAM
* 10.1" 1280x800 LED display
* Mali T860MP4 GPU
* 16GB eMMC
* 38 Whrs battery
* 2x USB 3.1 Type-C ports
* 1x USB 2.0 Type-A port

_Goal: run Debian on the ChromeOS_

The goal was achieved working progresively:

1. Dual boot the ChromeOS running archlinux on a SD Card
2. Replace the archlinux filesystem with a Debian rootfs 
3. Replace the kernel (ChromeOS) with the mainstream Linux kernel
4. Troubleshooting: modify kernel options and install firmware
5. Replace the kernel (Linux) with the latest Debian kernel 

## 1. Archlinux

Goal: try out a working linux distribution on the Chromebook

[Arch Linux](https://archlinuxarm.org/platforms/armv8/rockchip/asus-chromebook-flip-c101palinuxarch)

_On the chromebook_

This links provides an excellent overview of all chromebook hardware features and instructions for the installation of Arch Linux from a SD Card in dual boot setup. Closely follow the instructions in order to:
* Enable Developer Mode
* Enable booting from USB
* Partition the SD card
* Download the tarball. The Arch Linux tarball includes the ChromeOS kernel and an Arch Linux filesystem. 
* Copy the file system to the second partition on the card
* Flash the kernel into the first partition on the card

## 2. Debian Root File System

Goal: running Debian system on the SD Card booting the ChromeOS kernel provided by archlinux and a debian filesystem. 

_On the laptop_

Insert the SD card and mount the second partition of the card - the one which will host the rootfs:
```bash
mkdir rootfs
sudo mount /dev/sdb2 rootfs
```

Use debootstrap in order to create a Debian sid root file system:
```bash
sudo debootstrap --arch=arm64 --verbose --foreign sid rootfs
```
Changeroot to the new created filesystem:
```bash
sudo chroot
```

Create a user and set the user's and the root passwords:
```bash
passwd
adduser eramon
passwd eramon
```

Install following packages:
```bash
apt-get install wicd-curses openssh-server sudo git xserver-xorg gnome
```

Exit the chroot, umount the filesystem and sync:
```bash
exit
umount rootfs
sync
```
_ Back on the chromebook_
Insert the SD Card on the chromebook and press CTRL-U as soon as the boot screen appears.

The firmware files and the modules will be missing on the system. Get them from the archlinux tarball:
```bash
sudo cp -r rootfs/lib/firmware /lib/
sudo cp -r rootfs/lib/modules/* /lib/modules/
```
The module files in archlinux seem to be compressed: they have the extension .ko.gz. In order for them to work with the debian filesystem, it's necessary to unzip them first:
```bash
TODO
```

## 3. Mainstream Linux Kernel 
Goal: replace the ChromeOS kernel through the mainstream linux kernel. 

As a reference, I followed the instructions on the archlinux PKGBUILD file for linux-gru:
https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-gru/PKGBUILD   

In order to avoid cross compiling issues and to keep things simple, I aimed to do all the compiling and even the flashing directly on the chromebook. At this stage I had already  working environment consisting on the chromeos kernel and the new debian filesystem.   

First generate a default configuration: 
```bash
make defconfig
```
Compile the kernel:
```bash
make Image Dtbs Modules
```
TODO:
- FDT and .its file
- Link to the PKGBUILD from archlinux
- mkimage
- vbutil_kernel 

Flash the new generated image to the first partition of the SD card, sync and reboot:
```bash
sudo dd if=vmlinuz.kpart of=/dev/mmcblk1p1
sudo sync
sudo reboot
```

In case the screen remains blank when trying to boot the new kernel: do serial port debugging with the help of a Suzy Cable:
* Use the USB-C port close to the screen, since the other one does not offer a serial port connection. 
* Make sure the cable is plugged in the right position (the letters DBG have to be upside).
* Use minicom to connect via serial port and follow the boot process

## 4. Troubleshooting

Goal: modify the kernel configuration as needed in order to have all hardware working. Install missing firmware. 

After a first test run - compiling and flashing the kernel with the default configuration - following flaws could be identified:
- The wireless was not working
- The touchscreen was not working
- The touchpad was not working
- The virtual terminals were not working: only the X
- The sound was not working

### Wireless
A look to dmesg revealed that the firmware files were missing:  
```bash
```

Comparing with the running ChromeOS system on the same chromebook, I found out which firmware files were missing and which debian package will provide them: firmware-libertas. 
```bash
apt-get install firmware-libertas
```
_Note: in order to install this firmware the contrib non-free sources must be included in /etc/apt/sources.list._

### Touchscreen & Touchpad
In order for the touchscreen and the touchpad to work, we need to manually add the missing kernel modules to the kernel configuration.

Edit .config to include the following devices as kernel modules:
```bash
CONFIG_MOUSE_ELAN_I2C=m
CONFIG_MOUSE_ELAN_I2C_I2C=m
CONFIG_TOUCHSCREEN_ELAN=m
```
After this, we see the elan touchpad in /proc/bus/input/devices. The touchpad is working.

### Sound
The sound wasn't working, although the corresponding modules seemed to be in place after comparing with a running ChromeOS system on the same device. 

TODO Include dmesg output and name of the sound modules

In addition, the sound settings in gnome only showed a "dummy output" as sound device. Apparently Debian does not include users in the audio group by default, so adding the user to the group improved the situation:
```bash
adduser eramon audio 
```

Although the sound wasn't still working, I was able to see two proper output devices in the sound settings in gnome. 

_At the time of this writing, the sound is still not working._

### Further issues
There is some other issues I did not go into:
* There is no support for virtual terminals or it is not working. Ctrl-Alt-Fn shows a blank screen. Luckily, X are working. 
* The system seems to recover well from suspend when it suspends in its own (after no pressing any key for a while). However, if closing the lid manually, the system is not able to recover. 
* The sound, brightness and other such keys (the top row on the Asus Chromebook keyboard) does not work as intended. Probably this is just some configuration missing.

## 5. Debian Kernel  

Goal: replace the mainstream linux kernel through the latest debian kernel. 
Tasks:
* Modify the kernel in order to include the modules for the touchpad and touchscreen to work
* Modify the FDT image in order to include an initramfs since the debian kernel can't boot without it
* In order for the device to be supported in the future, submit a bug to the debian kernel team in order to suggest including the missing modules.

Clone the kernel source code:
```bash
git clone https://salsa.debian.org/kernel-team/linux.git
```

## 6. Internal Flash

## 7. Legacy Boot 
* U-Boot?
* Depthcharge?  

Legacy boot on the Chromebooks is used today to boot Windows in the most powerful devices (for example the Pixelbook). It seems to be the right way to proceed to use this boot method in order to boot an arbitrary linux distribution by pressing Ctrl-L from the boot screen.

