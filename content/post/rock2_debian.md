+++
math = true
date = "2017-07-27T15:03:58+02:00"
title = "Debian on Radxa Rock Square"
tags = ["arm","debian","u-boot"]
highlight = true

[header]
  caption = ""
  image = ""

+++

*Radxa Rock 2 Square is a Rockchip RK3288 based solution.*

[Available images](http://dl.radxa.com/rock2/images/e)

The available images run with:

* A modified version of u-boot
* The Rabian operating system (which is based on Debian 8)
* A custom radxa kernel.

The goal is to get a pure Debian distribution up and running on the board:

1. Debian u-boot
2. Debian armhf linux-image kernel
3. Pure Debian root file system

For this small project, we have two Radxa Rock 2 Square boards:

1. The first one used for development, on which we aim to run Debian, booting the Radxa kernel with the Radxa u-boot. We'll run using the internal flash storage of the board. This will be the _Development Board_.
2. The second one used as testing platform. The goal is to be able to boot an official Debian kernel on this one using the official Debian u-boot and run an unmodified Debian sid on it. We'll boot from a SD card. This will be the _Integration Board_.

We also need a SD card with a capacity of 8 gigabytes at least.

## Try out a debian-based distribution

_Development Board_

Connect the board to a keyboard and a monitor or use the serial port of the board and minicom from your laptop to see the output of the board.

First of all, we will try an existing image to see how it works. The image we are using is a Debian-based one which will be flashed to the board internal storage. Flashing an existing image to the board with the instructions and flash tool provided by Radxa is straightforward, just follow the instructions on the website:
	
[_Flash the image_](http://wiki.radxa.com/Rock/flash_the_image)

In a nutshell:

1.	Download the Image:

	[_Rabian SD Image_](http://dl.radxa.com/rock2/images/rabian/emmc/rock2_square_rabian_150720_5d53487_emmc.img.xz)

2.	Download and unzip the Radxa flashing tool:

	[_Radxa Flashing Tool_](http://dl.radxa.com/rock/tools/linux/Linux_Upgrade_Tool_v1.21.zip)

3.	Boot the board in bootloader mode and flash the image:
```bash
sudo radxa/Linux_Upgrade_Tool_v1.21/upgrade_tool uf radxa/rock2_square_rabian_150720_5d53487_emmc.img
```

A better alternative is to use the rkflashtool which is available as a Debian package: 
```bash
sudo apt-get install rkflashtool
```

The board will reboot automatically after the image flash is completed. Attaching a HDMI monitor and a keyboard to the board you should be able to log in to the Rabian system. Both the username and the password are "rock2".

## Create Debian rootfs

_Development Board_

After having tried out Rabian on the Rock2, we aim to run a newer, unmodified version of Debian on the board. In order to go there, we need to replace the root filesystem through a self-built one.

We'll use the Debian utility debootstrap for building the rootfs. The easiest and safest way is to use the available Rabian system - before replacing the rootfs - for setting up a custom rootfs with debootstrap. 

NOTE: it is also possible to generate the rootfs on other platform, using debootstrap --first-stage on the host architecture and --second-state using qemu. However, trying to do so lead to an EXT3 journal during the booting process. The reason is that the mkfs version used for creating the file system was much newer than the radxa kernel, which at the time of this writting was already quite old.

1.	Boot the board and log in to the system:

	```
	username: rock2
	password:
	```
2.	Create the file which will host the root file system:
	```	
	dd if=/dev/zero of=debian.ext4 bs=1M count=4096
	```

3.	Create an ext4 filesystem on the file:
	```
	sudo mkfs.ext4 debian.ext4
	```

	The execution of the mkfs command is actually the reason for creating the rootfs file on the Rock2 directly. Creating the file system on another host which a newer kernel can lead to a journal error when trying to boot the system afterwards.

4.	Mount the file as a loop device:
	```
	sudo mount -o loop rootfs.ext4 /mnt
	```

5.	Use Debootstrap to download a Debian sid to the mount point: 
	```
	debootstrap sid /mnt
	```

6.	Copy the kernel modules from the running Rabian system:
	```
	sudo mkdir -p /mnt/lib/modules
	sudo cp -r /lib/modules/* /mnt/lib/modules

	```

7.	Change root to the new filesystem:
	```
	sudo chroot /mnt
	```

	Using sudo, allow root rights to the new created user:
	```
	visudo
	```

8.	Add root mount point to fstab:
	```shell
	sudo echo "/dev/mmcblk0p2 / ext4 relatime,errors=remount-ro 0 1" > /etc/fstab
	```

9. 	Do some fine tuning on the new filesystem. 

	* Create a non-root user (and set his password):
	```
	adduser eramon
	```
	
	* Install some packages:
	```
	apt-get install sudo, openssh-server, locales
	```
	Of course further installation of packages can follow later after we finally boot the system.j

	* Modify /etc/network/interfaces in order for the new system to get connectivity automatically

	* Set the hostname:
	``` 
	echo absinthe > /etc/hostname
	```
	* Add the following line to  /etc/hosts:
	```
	127.0.0.1	absinthe
	```

The Debian filesystem is now ready. Umount the file:
```
sudo umount /mnt
```

Copy the file to your laptop or computer. We'll use it twice afterwards:

1.	Flash the filesystem on the development board
2.	Copy the rootfs to the second partition of the sdcard

## Flash the filesystem

_Development Board_

The default rootfs (Rabian) provided with the complete image we flashed at the beginning will be now replaced by our own Debian rootfs. In order to do that:

1.	Put the board into bootloader mode (same as you did before when flashing the complete image)
2.	Use the radxa flashtool for flashing the rootfs into the correct place in the internal memory of the board:

	```
	sudo radxa/Linux_Upgrade_Tool_v1.21/upgrade_tool di linuxroot debian.ext4
	```
Reboot the board and log in with the credentials you set before.


## Prepare the SD Card

_On your laptop_ 

### Partitions

The SD card partition scheme must be as follows:

* One 1G partition for /boot
* One 5G partition for rootfs

Insert the SD card and check the device name using:
```
dmesg
```
or
```
cat /proc/partitions
```

In these examples we asume /dev/mmcblk1 to be the SD card.

Use fdisk to create the partitions as explained above:
```
sudo fdisk /dev/sdb
```

After partitioning the SD, create an EXT2 filesystem on the boot partition:
```
mkfs.ext2 /dev/sdb2
sudo sync
```
### Rootfs Partition

Using dd, copy the rootfs partition created before to the sdcard:

```
dd if=rootfs.ext4 of=/dev/sdb2
sudo sync
```

### Boot Partition

_Development Board_
 
Insert the sdcard in the development board. Mount the first (boot) partition at /boot:
```
sudo mount /dev/mmcblk1p1 /boot
```

For the generation of the boot files, we will use flash-kernel. flash-kernel is a script which will put the kernel and initramfs in the boot location of embedded devices that don't load the kernel and initramfs directly from /boot.

The Radxa Rock2 was not supported by flash-kernel. I submited a patch within a bug report:
[flash-kernel bug report](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=868166)

Since the linux kernel did support the board, fixing this was quite straightforward. The patch consisted on adding an entry for the Rock2 to /usr/share/flash-kernel/db/all/db:
```
Machine: Radxa Rock 2 Square
Kernel-Flavors: armmp armmp-lpae
DTB-Id: rk3288-rock2-square.dtb
Boot-Script-Path: /boot/boot.scr
U-Boot-Script-Name: bootscr.uboot-generic
Required-Packages: u-boot-tools
```

Install flash-kernel:
```
sudo apt-get install flash-kernel
```

Install the debian kernel:
```
apt-cache search linux-image
sudo apt-get install linux-image-4.11.0-2-armmp
```

The installation of the kernel will copy the kernel image and the initramfs on /boot. In addition to this, the installation process will invoke flash-kernel which will install the boot script and the device tree binary on the same location:
```
eramon@absinthe:/boot$ ls
boot.scr               dtb-4.11.0-2-armmp         System.map-4.11.0-2-armmp
config-4.11.0-2-armmp  dtbs                       vmlinuz-4.11.0-2-armmp
dtb                    initrd.img-4.11.0-2-armmp
```
Umount the sdcard partition:
```
sudo umount /boot
sudo sync
```

### Debian's U-Boot

Download Debian u-boot from GitHub:
```
git clone git://anonscm.debian.org/git/collab-maint/u-boot.git
```

Configure and build:
```
make rock2_defconfig
make
```
After succesfully building the source, two binaries were generated:

* u-boot.bin 
* spl/u-boot-spl.bin

The Rockchip SoCs bootrom needs u-boot to be flashed to the SD card:

* In a special format called IDB
* With offset 0x40 block

In order to generate the IDB image, use the mkimage tool provided by u-boot:
```
tools/mkimage -n rk3288 -T rksd -d spl/u-boot-spl.bin rk3288_idb.img
```

For the rock2 board, the kernel setting CONFIG_ROCKCHIP_SPL_BACK_TO_BROM is by default set to 1. This means that the u-boot binary must be appended to the IDB image:
```
cat u-boot.bin >> rk3288_idb.img
```

The complete image must be now flashed to offset 0x40 on the SD card (e.g. seek=64 as dd parameter):
```
dd if=rk3288_idb.img of=/dev/mmcblk1 seek=64
```

Although this way worked, the right approach is to have a debian package supporting the board.

_TODO u-boot patch for Rock2 support_

## Boot from the SD Card

The sdcard is now ready. In a nutshell, we did the following:

1.	Generating and installing all necessary boot files on the development board, with the first sdcard partition mounted on /boot
2. 	Creating a debian file system and copying it to the second sdcard partition
3. 	Building u-boot and copying the files to offset 0x40 on the sdcard.

_Integration Board_

The board is able to boot both from internal memory and from a SD card: if a SD card is inserted, it will check first there for a bootable system. However, this is not true for u-boot: even if the SD card is equiped with a running u-boot, the board will always boot from the u-boot stored on the internal memory. In order to solve this, allowing the rock2 to use the u-boot on the SD card, we need to "brick" the board, zeroing the memory position where u-boot is located.

I found some useful information [here.](http://git.denx.de/?p=u-boot.git;a=blob;f=doc/README.rockchip;h=e0572c80b9c7d6e22bfc695d0d73fde74f6789d5;hb=df61a74e6845ec9bdcdd48d2aff5e9c2c6debeaa)

The firefly board refered to this README has the same architecture as the Rock2. Among other things, on this README is described how to remove the existing u-boot from the board, thus allowing to boot from a custom u-boot on the SD card:
```
dd if=/dev/zero of=/dev/mmcblk0 bs=1M
```

After the location where u-boot is stored in the internal flash of the board is zeroised, the board won't be able to boot from there anymore, trying to find a suitable u-boot on the sdcard. 

Connect your laptop to the serial port of the board and use minicom to see the output: 
```
sudo minicom /dev/ttyUSB0
```

Insert the sdcard and reboot the integration board. Debian u-boot will boot the debian kernel and the debian system will start.

```
Debian GNU/Linux buster/sid absinthe ttyS2

absinthe login: 

```

Done :)


