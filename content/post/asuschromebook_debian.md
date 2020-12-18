+++
math = true
date = "2018-12-04T10:00:00+02:00"
title = "Debian on Asus Chromebook Flip C101PA"
tags = ["debian","arm"]
highlight = true

[header]
  caption = ""
  image = ""

+++

*The Asus Chromebook Flip C101PA (bob) is a convertible touchscreen laptop powered by an ARMv8 Rockchip RK3399 hexa-core processor and 4GB RAM, measuring 10.4" x 7.2" x 0.6" and weighing 1 kg.*

Features:

* Rockchip RK3399 (OP1) dual-core 2.0GHz Cortex-A72 and quad-core 1.4GHz Cortex-A53 processor
* 4GB LPDDR3 RAM
* 10.1" 1280x800 LED display
* Mali T860MP4 GPU
* 16GB eMMC
* 38 Whrs battery
* 2x USB 3.1 Type-C ports
* 1x USB 2.0 Type-A port

ARM Chromebooks ship with the Coreboot bootloader and use Depthcharge as the payload. Further reading:

[Coreboot Website](https://www.coreboot.org)

[Depthcharge Source Code](https://github.com/coreboot/depthcharge)

The kernel is signed and packed in a custom format and must be flashed to a dedicated partition on the sdcard or to the internal ssd. Further reading:

[Chromium: Disk Format](https://www.chromium.org/chromium-os/chromiumos-design-docs/disk-format)

After enabling Developer Mode and USB Boot, it's possible to boot either ChromeOS from the internal disk of the chromebook (Ctrl-D) or from USB (Ctrl-U).

__Goal: run Debian on the ChromeOS__

__Motivation:__ the Asus C101PA is a light (and cheap) laptop - perfect to carry it everywhere. Using ChromeOS only was not an option for me, so I aimed to run Debian Linux on it.   

__Milestones:__

1. Dual boot ChromeOS and Linux: run archlinux on a sdcard
2. Replace the archlinux filesystem with a Debian rootfs 
3. Replace the kernel (ChromeOS) with the mainline Linux kernel
4. Troubleshooting: modify kernel options and install firmware
5. Flash the working kernel to the chromebook internal memory
5. Replace the kernel (Linux) with the latest Debian kernel 

Files and scripts available in github:

[https://github.com/eramons/chromebook](https://github.com/eramons/chromebook)

## 1. Archlinux

__Goal: try out a working linux distribution on the Chromebook__

_On the chromebook_

I started taking a look at the excelent Arch Linux wiki:
[ArchLinuxArm](https://archlinuxarm.org/platforms/armv8/rockchip/asus-chromebook-flip-c101pa)

This links provides instructions for the installation of Arch Linux from a sdcard in dual boot setup. I closely followed the instructions in order to:

* Enable Developer Mode
* Enable booting from USB
* Partition the sdcard
* Download the tarball. The Arch Linux tarball includes the ChromeOS kernel and an Arch Linux filesystem. 
* Copy the file system to the second partition on the card
* Flash the kernel into the first partition on the card

## 2. Debian Root File System

__Goal: running Debian system on the sdcard booting the ChromeOS kernel provided by archlinux and a debian filesystem.__

For the preparation of the debian rootfs, I inserted the sdcard in my laptop and mounted the partition which would host the rootfs. I used then debootstrap to install the base system.

_On the laptop_

Mount the second sdcard partition:
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
_Back on the chromebook_

Insert the sdcard on the chromebook and press CTRL-U as soon as the boot screen appears.

The firmware files and the modules will be missing on the system. Get them from the archlinux tarball:
```bash
sudo cp -r rootfs/lib/firmware /lib/
sudo cp -r rootfs/lib/modules/* /lib/modules/
```
The module files in archlinux seem to be compressed: they have the extension .ko.gz. In order for them to work with the debian filesystem, I had to unzip them first.

## 3. Mainline Linux Kernel 
__Goal: replace the ChromeOS kernel through the mainline linux kernel.__

As a reference, I read the instructions for building a package on the archlinux PKGBUILD file for linux-gru in order to figure out which steps needed to be done:
[archlinuxarm/PKGBUILDS](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-gru/PKGBUILD) 

In order to avoid cross compiling issues and to keep things simple, I aimed to do all the compiling and even the flashing directly on the chromebook. At this stage I had already a working environment consisting on the chromeos kernel and the new debian filesystem.   

First get the mainline kernel source code:
```
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git linux-git
```

Get into the source directory and generate a default configuration: 
```bash
cd linux-git
make defconfig
```
Compile the kernel:
```bash
make Image Dtbs Modules
```
Download the kernel.its file used by arch linux to build their package:
[kernel.its](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-gru/kernel.its)

There are several configurations in this file, we only need the first one. We also need to modify the path and name of the dtbs file. 

I modified the file as follows:

[https://github.com/eramons/chromebook/mainlinekernel.its](https://github.com/eramons/chromebook/mainlinekernel.its)

```
/dts-v1/;

/ {
    description = "Linux mainline kernel image with one blob";
    images {
        kernel@1{
            description = "kernel";
            data = /incbin/("linux-git/arch/arm64/boot/Image");
            type = "kernel_noload";
            arch = "arm64";
            os = "linux";
            compression = "none";
            load = <0>;
            entry = <0>;
        };
        fdt@1{
            description = "rk3399-gru-bob.dtb";
            data = /incbin/("linux-git/arch/arm64/boot/dts/rockchip/rk3399-gru-bob.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash@1{
                algo = "sha1";
            };
        };
    };
    configurations {
        default = "conf@1";
        conf@1{
            kernel = "kernel@1";
            fdt = "fdt@1";
        };
    };
};
```

Make image:
```
mkimage -D "-I dts -O dtb -p 2048" -f mainlinekernel.its vmlinux.uimg
```

Prepare the cmdline:
```
echo "console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8 console=tty1 init=/sbin/init root=PARTUUID=%U/PARTNROFF=1 rootwait rw noinitrd loglevel=4" > cmdline_mainline
```

Generate an empty bootloader.bin file:
```
dd if=/dev/zero of=bootloader.bin bs=512 count=1
```

Run vbutil in order to generate a boot image for the chromebook:
```
vbutil_kernel
        --pack vmlinux.kpart
        --version 1
        --vmlinuz vmlinux.uimg
        --arch aarch64
        --keyblock /usr/share/vboot/devkeys/kernel.keyblock
        --signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk
        --config cmdline_mainline
        --bootloader bootloader.bin
```

Flash the new generated image to the first partition of the sdcard, sync and reboot:
```bash
sudo dd if=vmlinuz.kpart of=/dev/mmcblk1p1
sudo sync
sudo reboot
```

My script automating the aforementioned steps (mkimage, cmdline, vbutil) is available here:

[https://github.com/eramons/chromebook/blob/master/run.sh](https://github.com/eramons/chromebook/blob/master/run.sh)

In case the screen remains blank when trying to boot the new kernel it helps to do serial port debugging with the help of a SuzyQable:
[SuzyQable](https://www.chromium.org/chromium-os/ccd#TOC-SuzyQ-SuzyQable)

* Use the USB-C port close to the screen, since the other one does not offer a serial port connection. 
* Make sure the cable is plugged in the right position (the letters DBG have to be upside).
* Use minicom to connect via serial port and follow the boot process

## 4. Troubleshooting

__Goal: modify the kernel configuration as needed in order to have all hardware working. Install missing firmware.__ 

After a first test run - compiling and flashing the kernel with the default configuration - I identified several flaws:

- The wireless was not working
- The touchscreen was not working
- The touchpad was not working
- The sound was not working

### Wireless
A look to dmesg revealed that the problem was just that the firmware files were missing. 

Comparing with the running ChromeOS system on the same chromebook, I found out which firmware files were missing and which debian package will provide them:  
```bash
apt-get install firmware-libertas
```
_Notes:_
 
* _In order to install this firmware the contrib non-free sources must be included in /etc/apt/sources.list_
* _Before getting the wi-fi to work, I used an ethernet cable and an usb-to-ethernet adapter_


After installing the firmware package wireless networking worked perfectly.

### Touchscreen & Touchpad
In order for the touchscreen and the touchpad to work, we need to manually add the missing kernel modules to the kernel configuration and re-compile.

Edit .config to include the following devices as kernel modules:
```bash
CONFIG_MOUSE_ELAN_I2C=m
CONFIG_MOUSE_ELAN_I2C_I2C=m
CONFIG_TOUCHSCREEN_ELAN=m
```

Re-build the kernel with make, make a new version with mkimage and generate a chromebook bootable image with vbutil_kernel, same as before.

After this, we see the elan touchpad in /proc/bus/input/devices. The touchpad is working.

### Sound
The sound wasn't working, although the corresponding modules seemed to be in place after comparing with a running ChromeOS system on the same device. 

In addition, the sound settings in gnome only showed a "dummy output" as sound device. Apparently Debian does not include users in the audio group by default, so adding the user to the group improved the situation:
```bash
adduser eramon audio 
```

Although the sound wasn't still working, I was able to see two proper output devices in the sound settings in gnome. 

_At the time of this writing, the sound is still not working._

### Further issues
There is some other issues I did not go into:

* The system seems to recover well from suspend when it suspends in its own (after no pressing any key for a while). However, if closing the lid manually, the system is not able to recover. 
* The sound, brightness and other such keys (the top row on the Asus Chromebook keyboard) does not work as intended. Maybe just a configuration issue.

## 5. Kernel and rootfs on the chromebook disk 

__Goal: Instead of booting from the sdcard, boot from the chromebook internal ssd__

Since we'll need several tries in order for the custom built kernel to boot, it's good to be able to boot both from the sdcard and from the internal drive. With Ctrl-D we'll be able to boot the experimental kernel we are building and with Ctrl-U we'll always be able to boot the already working mainline kernel on the sdcard first partition. 

Tasks:

* Use gparted to resize the stateful partition
* Create a new KERN-D and ROOT-D partitions and change the boot priorities
* Flash the kernel to the KERN-D partition
* Install a base debian system on ROOT-D using debootstrap

Consequence: ChromeOS won't boot anymore. The reason is that resizing a partition changes the boot priority of the KERN partitions on the same disk as the resized partition.

If you plan to restore ChromeOS at some point, run:

```
 cgpt show /dev/mmcblk1
```

Save the output. In order to have ChromeOS to boot again, the values of this fields must be set as they were originally (using cgpt). 
For example, the following command will restore the settings of KERN-A:
```
sudo cgpt add -i 2 -S 1 -T 0 -P 2 /dev/mmcblk1
```

Note that after doing this, it won't be possible to boot debian anymore with Ctrl-D. Unfortunately coreboot/depthcharge only allow to boot one kernel (with Ctrl-D). 

Original settings (output of cgpt show):

* KERN-A: priority=2 tries=0 successful=1
* KERN-B: priority=1 tries=0 successful=1
* KERN-C: priority=0 tries=15 successful=0


So I resized the so called "stateful partition" from 10.53 GB to 5.00 GB:
```
     8671232    10485760       1  Label: "STATE"
                                  Type: Linux data
                                  UUID: 1F4D5818-8E6B-0746-B1F9-E2E206777C85
```

I created two new partitions:

* For my debian kernel: KERN-D with size 100 MB
* For my debian rootfs: ROOT-D with size 9.45 GB (e.g. all unallocated space after resize STATE and creating KERN-D)

```
    30539776      204800      13  Label: "KERN-D"
                                  Type: ChromeOS kernel
                                  UUID: 9CAA153C-8A88-0A4D-B750-FA2F52FB3A2E
                                  Attr: priority=10 tries=5 successful=1 
    10719232    19820544      14  Label: "ROOT-D"
                                  Type: 0FC63DAF-8483-4772-8E79-3D69D8477DE4
                                  UUID: C5E4E377-6D8F-4747-AA1F-6A8EEDDF031A
                                  UUID: C5E4E377-6D8F-4747-AA1F-6A8EEDDF031A
```

Set the priority, tries and succesful flag for the KERN-D partition:
```
sudo cgpt add -i 1 -S 1 -T 0 -P 2 /dev/mmcblk0
```

Use debootstrap to create a debian rootfs on the new ROOT-D:
```
sudo mount /dev/mmcblk1p14 dev/mnt
sudo debootstrap sid dev/mnt
sudo chroot dev/mnt
```

After chroot-ing to the new filesystem, I set the hostname, the root password, created an user and installed some useful packages:

```
passwd
adduser eramon
cat "chupito" > /etc/hostname
apt-get install locales wicd-curses sudo
dpkg-reconfigure locales
visudo
exit
sudo sync
```

To test the new setup is working, I flashed the working mainline kernel to KERN-D. For that, I modified the cmdline in order to use ROOT-D as the rootfs. 
```
cat "console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8 console=tty0 init=/sbin/init root=PARTUUID=c5e4e377-6d8f-4747-aa1f-6a8eeddf031a rootwait rw noinitrd loglevel=4" > cmdline
```

_NOTE: the PARTUUID of the rootfs partition is hardcoded - it shouldn't but I did not know how to do otherwise_

To find out the PARTUUID of ROOT-D:
```
ls -l /dev/disk/by-partuuid/
```

Run vbutil as before using this modified cmdline:
```
vbutil_kernel 
	--pack vmlinux.kpart 
	--version 1 
	--vmlinuz vmlinux.uimg 
	--arch aarch64 
	--keyblock /usr/share/vboot/devkeys/kernel.keyblock 
	--signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk 
	--config cmdline 
	--bootloader bootloader.bin
``` 

Flash the modified kernel image to the KERN-D partition and reboot:
```
sudo dd if=vmlinux.kpart of=/dev/mmcblk1p13
sudo sync
sudo reboot
```

After doing this, I was able to boot my custom mainline kernel with Ctrl-D. Once logged in over serial, we can install the X system and other necessary packages:
```
apt-get install xserver-xorg gnome firmware-libertas
``` 

_NOTE I: same as before and before getting the wi-fi to work, I had to use a network cable._ 

_NOTE II: the chromebook has only a little drive (16 GB). For a productive setup, I ended up moving my /home partition to the sdcard - letting only the base system / on the ROOT-D partition._  


## 6. Debian Kernel  

__Goal: replace the mainline linux kernel through the latest debian kernel.__ 

Tasks:

* Get the latest kernel source
* Modify the kernel in order to include the modules for the touchpad and touchscreen to work
* Build the debian package and install it
* Modify the FDT image in order to include an initramfs since the debian kernel can't boot without it
* Build the kernel image

Instructions for re-building a debian kernel package (see "Building a custom kernel from Debian kernel source"):

[Rebuilding official Debian kernel packages](https://kernel-team.pages.debian.net/kernel-handbook/ch-common-tasks.html#s-common-official)


Get the kernel source code:
```bash
apt-get install linux-source-4.19
tar xaf /usr/src/linux-source-4.19.16.tar.xz
```

Install following packages:
```
apt-get install build-essential, fakeroot, build-dep, devscripts
```

Run the following command in order to generate the default configuration:
```
make defconfig
```

Edit .config in order to include the necessary kernel modules we found out in the "Troubleshooting" section:
```
vi .config

CONFIG_MOUSE_PS2_ELANTECH=m
CONFIG_MOUSE_ELAN_I2C=m
CONFIG_TOUCHSCREEN_ELAN=m
```

_TODO: In order for the device to be supported in the future, submit a bug to the debian kernel team in order to suggest including the missing modules._

Build the kernel package:
```
make deb-pkg
```

Install following file - generated by a succesfull build:
```
dpkg -i linux-image-4.19.16-1_arm64.deb
```

This will install -among others- the following files:
```
/boot/initrd-img-4.19.16
/boot/dtbs/rockchip/rk3399-gru-bob.dtb
```

Chromebooks don't use a ramdisk for booting. For the mainline kernel we used before, we didn't use one either. However in order for Debian to boot, a initramfs (or ramdisk) is mandatory. 

Get the files we need for making the image and which will be referenced in kernel.its:

* kernel: get the Image file from the compiled source: arch/arm64/boot/Image
* ramdisk: /boot/initrd-img-4.19.16 installed by dpkg before
* dtb: arch/arm64/boot/dts/rockchip/rk3399-gru-bob.dtb (also available under /boot/dtbs/rockchip/rk3399-gru-bob.dtb installed by dpkg before) 

Edit the kernel.its file we used before in order to include a ramdisk:
[https://github.com/eramons/chromebook/blob/master/debkernel.its](https://github.com/eramons/chromebook/blob/master/debkernel.its)

```
/dts-v1/;

/ {
    description = "Debian kernel image with one blob for the Asus C101 (Bob) and initramfs";

    images {
        kernel@1{
            description = "kernel";
            data = /incbin/("linux/debian/arch/arm64/boot/Image");
            type = "kernel_noload";
            arch = "arm64";
            os = "linux";
            compression = "none";
            load = <0>;
            entry = <0>;
        };
        fdt@1{
            description = "rk3399-gru-bob.dtb";
            data = /incbin/("linux/debian/arch/arm64/boot/dts/rockchip/rk3399-gru-bob.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash@1{
                algo = "sha1";
            };
        };
	ramdisk@1 {
           description = "initramfs";
           data = /incbin/("/boot/initrd.img-4.19.16");
           type = "ramdisk";
           arch = "arm64";
           compression = "none";
           hash@1 {
                algo = "sha1";
           };
	};
    };
    configurations {
        default = "conf@1";
        conf@1{
            kernel = "kernel@1";
            fdt = "fdt@1";
            ramdisk = "ramdisk@1";
        };
    };
};
```

Make image:
```
mkimage -D "-I dts -O dtb -p 2048" -f debkernel.its vmlinux.uimg
```

Find out the partition UID of the root partition:
```
ls -l /dev/disk/by-partuuid/
```

Prepare the cmdline:
```
echo "console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8 console=tty0 init=/sbin/init root=PARTUUID=c5e4e377-6d8f-4747-aa1f-6a8eeddf031a rootwait rw loglevel=4" > cmdline_debian
```

Generate an empty bootloader.bin file:
```
dd if=/dev/zero of=bootloader.bin bs=512 count=1
```

Run vbutil in order to generate a boot image for the chromebook:
```
vbutil_kernel 
	--pack vmlinux.kpart 
	--version 1 
	--vmlinuz vmlinux.uimg 
	--arch aarch64 
	--keyblock /usr/share/vboot/devkeys/kernel.keyblock 
	--signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk 
	--config cmdline_debian 
	--bootloader bootloader.bin
```

Flash the new generated image to the ROOT-D partition on the chromebook, sync and reboot:
```bash
sudo dd if=vmlinuz.kpart of=/dev/mmcblk1p14
sudo sync
sudo reboot
```

Done :) Debian on the chromebook: a debian rootfs and a customized, self-built debian kernel. 

_And now? Next steps:_

* Coreboot/Depthcharge: build an u-boot payload for the Asus C101. Extend flash-kernel to support the board.  
* Alternatively, build a second depthcharge payload loaded through the legacy mechanism provided by the chromebooks (Ctrl-L) which loads the KERN-D kernel. 

