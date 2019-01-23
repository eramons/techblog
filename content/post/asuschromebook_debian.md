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

__Goal: run Debian on the ChromeOS__

The goal was achieved working progresively:

1. Dual boot the ChromeOS running archlinux on a SD Card
2. Replace the archlinux filesystem with a Debian rootfs 
3. Replace the kernel (ChromeOS) with the mainstream Linux kernel
4. Troubleshooting: modify kernel options and install firmware
5. Flash the working kernel to the internal memory of the chromebook
5. Replace the kernel (Linux) with the latest Debian kernel 

## 1. Archlinux

__Goal: try out a working linux distribution on the Chromebook__

_On the chromebook_

Start taking a look at the excelent Arch Linux wiki:
[ArchLinuxArm](https://archlinuxarm.org/platforms/armv8/rockchip/asus-chromebook-flip-c101palinuxarch)

This links provides instructions for the installation of Arch Linux from a SD Card in dual boot setup. Follow the instructions in order to:

* Enable Developer Mode
* Enable booting from USB
* Partition the SD card
* Download the tarball. The Arch Linux tarball includes the ChromeOS kernel and an Arch Linux filesystem. 
* Copy the file system to the second partition on the card
* Flash the kernel into the first partition on the card

## 2. Debian Root File System

__Goal: running Debian system on the SD Card booting the ChromeOS kernel provided by archlinux and a debian filesystem.__

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
_Back on the chromebook_

Insert the SD Card on the chromebook and press CTRL-U as soon as the boot screen appears.

The firmware files and the modules will be missing on the system. Get them from the archlinux tarball:
```bash
sudo cp -r rootfs/lib/firmware /lib/
sudo cp -r rootfs/lib/modules/* /lib/modules/
```
The module files in archlinux seem to be compressed: they have the extension .ko.gz. In order for them to work with the debian filesystem, it's necessary to unzip them first:
```bash
TODO unzip 
```

## 3. Mainstream Linux Kernel 
__Goal: replace the ChromeOS kernel through the mainstream linux kernel.__

As a reference, I read the instructions for building a package on the archlinux PKGBUILD file for linux-gru in order to figure out which steps needed to be done:
[archlinuxarm/PKGBUILDS](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-gru/PKGBUILD) 

In order to avoid cross compiling issues and to keep things simple, I aimed to do all the compiling and even the flashing directly on the chromebook. At this stage I had already a working environment consisting on the chromeos kernel and the new debian filesystem.   

First generate a default configuration: 
```bash
make defconfig
```
Compile the kernel:
```bash
make Image Dtbs Modules
```
Download the kernel.its file used by arch linux to build their package:
[kernel.its](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-gru/kernel.its)

There are too many configuration in this file, we only need the first one. We also need to modify the path and name of the dtbs file. Modify the file as follows:
```
/dts-v1/;

/ {
    description = "Chrome OS kernel image with one FDT blob";
    images {
        kernel@1{
            description = "kernel";
            data = /incbin/("arch/arm64/boot/Image");
            type = "kernel_noload";
            arch = "arm64";
            os = "linux";
            compression = "none";
            load = <0>;
            entry = <0>;
        };
        fdt@1{
            description = "rk3399-gru-bob.dtb";
            data = /incbin/("arch/arm64/boot/dts/rockchip/rk3399-gru-bob.dtb");
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
```

Make image:
```
mkimage -D "-I dts -O dtb -p 2048" -f kernel.its vmlinux.uimg
```

Prepare the cmdline:
```
echo "console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8 console=tty1 init=/sbin/init root=PARTUUID=%U/PARTNROFF=1 rootwait rw noinitrd loglevel=4" > cmdline
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
        --config cmdline
        --bootloader bootloader.bin
```

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

__Goal: modify the kernel configuration as needed in order to have all hardware working. Install missing firmware.__ 

After a first test run - compiling and flashing the kernel with the default configuration - following flaws could be identified:

- The wireless was not working
- The touchscreen was not working
- The touchpad was not working
- The virtual terminals were not working: only the X
- The sound was not working

### Wireless
A look to dmesg revealed that the problem was just that the firmware files were missing. 

Comparing with the running ChromeOS system on the same chromebook, I found out which firmware files were missing and which debian package will provide them:  
```bash
apt-get install firmware-libertas
```
_Note: in order to install this firmware the contrib non-free sources must be included in /etc/apt/sources.list._

After installing the firmware package wireless networking worked perfectly.

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

## 5. Internal memory

__Goal: Instead of booting from the SDCard, boot from the chromebook internal memory__

Since we'll need several tries in order for the custom built kernel to boot, it's good to be able to boot both from the SDCard and from the internal memory. With Ctrl-D we'll be able to boot with the kernel we are building and with Ctrl-U we'll always be able to boot a working kernel from the SDCard. 

Consequence: ChromeOS won't boot anymore, although it's easy to restore it afterwards. 

* Use gparted to reduce the stateful partition
* Create a new KERN-D and ROOT-D partitions and change the boot priorities
* Flash the kernel to the KERN-D partition
* Install a base debian system on ROOT-D using debootstrap

TODO Detailed steps

## 6. Debian Kernel  

__Goal: replace the mainstream linux kernel through the latest debian kernel.__ 

Tasks:

* Get the latest kernel source
* Modify the kernel in order to include the modules for the touchpad and touchscreen to work
* Build the debian package and install it
* Modify the FDT image in order to include an initramfs since the debian kernel can't boot without it
* Build the kernel image

Get the kernel source code:
```bash
apt-get install linux-source-4.18
tar xaf /usr/src/linux-source-4.18.tar.xz
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

CONFIG_MOUSE_PS2_ELANTECH
CONFIG_MOUSE_ELAN_I2C
CONFIG_TOUCHSCREEN_ELAN
```

_TODO: In order for the device to be supported in the future, submit a bug to the debian kernel team in order to suggest including the missing modules._

Build the kernel package:
```
make deb-pkg
```

Install following file - generated by a succesfull build:
```
dpkg -i linux-image-4.18.20-1_arm64.deb
```

This will install -among others- the following files:
```
/boot/initrd-img-4.18.20
/boot/dtbs/rockchip/rk3399-gru-bob.dtb
```

Chromebooks don't use a ramdisk for booting. For the mainstream kernel we used before, we didn't use one either. However in order for Debian to boot, a initramfs (or ramdisk) is mandatory. 

Get the files we need for making the image and which will be referenced in kernel.its:

* kernel: get the Image file from the compiled source: arch/arm64/boot/Image
* ramdisk: /boot/initrd-img-4.18.20 installed by dpkg before
* dtb: /boot/dtbs/rockchip/rk3399-gru-bob.dtb installed by dpkg before 

Edit the kernel.its file we used before in order to include a ramdisk:
```
/dts-v1/;

/ {
    description = "Debian kernel image with one blob and initramfs";
    images {
        kernel@1{
            description = "kernel";
            data = /incbin/("Image");
            type = "kernel_noload";
            arch = "arm64";
            os = "linux";
            compression = "none";
            load = <0>;
            entry = <0>;
        };
        fdt@1{
            description = "rk3399-gru-bob.dtb";
            data = /incbin/("rk3399-gru-bob.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash@1{
                algo = "sha1";
            };
        };
	ramdisk@1 {
                description = "initramfs";
                data = /incbin/("boot/initrd.img-4.18.20");
                type = "ramdisk";
                arch = "arm64";
                os = "linux";
                compression = "gzip";
                load = <00000000>;
                entry = <00000000>;
                hash@1 {
                        algo = "sha1";
                };
	};
    };
    configurations {
        default = "conf@1";
        conf@1{
            kernel = "kernel@1";
            ramdisk = "ramdisk@1";
            fdt = "fdt@1";
        };
    };
};
```

Make image:
```
mkimage -D "-I dts -O dtb -p 2048" -f kernel.its vmlinux.uimg
```

Find out the partition UID of the root partition:
```
ls -l /dev/disk/by-partuuid/
```

Prepare the cmdline:
```
echo "console=ttyS2,115200n8 earlyprintk=ttyS2,115200n8 console=tty1 init=/sbin/init root=PARTUUID=c5e4e377-6d8f-4747-aa1f-6a8eeddf031a rootwait rw loglevel=4" > cmdline_debian
```

Generate an empty bootloader.bin file:
```
dd if=/dev/zero of=bootloader.bin
```

Run vbutil in order to generate a boot image for the chromebook:
```
vbutil_kernel 
	--pack vmlinux-4.kpart 
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

## 7. Legacy Boot 
* U-Boot?
* Depthcharge?  

Legacy boot on the Chromebooks is used today to boot Windows in the most powerful devices (for example the Pixelbook). It seems to be the right way to proceed to use this boot method in order to boot an arbitrary linux distribution by pressing Ctrl-L from the boot screen.

/Goal

