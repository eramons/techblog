+++
math = true
date = "2020-12-17T09:00:00+02:00"
title = "WIP Coreboot on ThinkPad W500"
tags = ["coreboot"]
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Give an old Lenovo Thinkpad W500 a second life featuring Coreboot and Ubuntu_

My partner replaced his laptop already two years and half ago, buying a beautiful brand-new Lenovo Thinkpad T480s. The old machine, a Lenovo Thinkpad W500 - heavier and bulkier and a lot less pretty - wasn't getting any further Windows updates and it was painfully slow, rarely being used. Furthermore, the battery was almost dead. 

I got permission to play around with this old machine - even after pointing out the risks - and to try to give it a second life:

 * Replacing the propietary firmware with a self-built Coreboot, since Thinkpads are usually good supported
 * Running the latest Ubuntu LTS Desktop version alongside the installed Windows distribution
 * Taking the battery away and using it plugged in only

_NOTE: the risks are a) bricking the machine b) breaking something during disassembling_

## Pre-conditions

 * The mainboard vendor and the mainboard model must be supported by Coreboot
 * To have a SPI flasher
 * To have a SPI pin clip
 * Find out the ROM size
 * Make a bootable Live Ubuntu USB stick

## 1. Preliminar research 

### 1.1. Vendor, board and chip

Googling a little around I found that the W500 was almost identical to T400, which was listed on the list of boards officially supported by coreboot. I also found the blogs of two guys which managed to install libreboot on the same laptop.

I own a SPI flasher (from the time I flashed Coreboot to my librem and I needed to have a brick-contingency plan). 

The flash chip on the mainboard of the W500 has 16 pins. I had a SPI pin clip, however it was a 8-pin only, so I had to order a new one.


### 1.2. ROM Size

To find out the ROM size, I booted the PC from a Ubuntu Live CD stick and ran the following:
```
ubuntu@ubuntu:~$ dmidecode | grep ROM\ Size
        ROM Size: 4096 kB
```

### 1.3 

Boot with Live Ubuntu USB stick.

Install flashrom. Flashrom is not available for ubuntu as a package, so I had to build it from code.

First install git and build dependencies>
```
ubuntu@ubuntu:~/flashrom$ sudo apt-get install git make gcc build-essential libcpi-dev libusb-1.0.0
```

Clone flashrom repository:
```
ubuntu@ubuntu:~/ git clone https://review.coreboot.org/flashrom.git
ubuntu@ubuntu:~/ cd flashrom
ubuntu@ubuntu:~/flashrom$ make
```

The generated flashrom binary can be found in the same directory, after succesfully build.
```
ubuntu@ubuntu:~/flashrom$ sudo cp flashrom /usr/local/bin/
ubuntu@ubuntu:~/flashrom$ sudo chmod +x /usr/local/bin/flashrom
```

Try to read something internally:
```
ubuntu@ubuntu:~/thinkpad500$ sudo flashrom -r w500_firmware_orig.rom -p internal
```

```
Found chipset "Intel ICH9M".
Enabling flash write... SPI Configuration is locked down.
FREG0: Flash Descriptor region (0x00000000-0x00000fff) is read-only.
FREG2: Management Engine region (0x00001000-0x001f5fff) is locked.
PR0: Warning: 0x003e0000-0x01ffffff is read-only.
At least some flash regions are read protected. You have to use a flash
layout and include only accessible regions. For write operations, you'll
additionally need the --noverify-all switch. See manpage for more details.
OK.
Found Programmer flash chip "Opaque flash chip" (4096 kB, Programmer-specific) mapped at physical address 0x0000000000000000.
Reading flash... Transaction error between offset 0x00001000 and 0x0000103f (= 0x00001000 + 63)!
Read operation failed!
FAILED.
```

I tried creating a layout.txt file:
```
ubuntu@ubuntu:~/thinkpad500$ cat layout.txt 
0x00000000:0x00000fff FREG0
```

And then:
```
ubuntu@ubuntu:~/thinkpad500$ sudo flashrom -p internal -l layout.txt -i FREG0 -r output.txt
```

## 2. Coreboot

Getting a SPI 16 pin clip was going to take a couple of weeks, so I started preparing the coreboot build. 

### 2.1. Payload

As payload, I would use SeeBIOS, which should allow:

 * First, still being able to boot the existing Windows installation
 * Second, boot Ubuntu after the installation 

SeeBIOS is the default payload downloaded and built during the coreboot build process.

### 2.2. Binary blobs

According to the documentation, for Intel boards following files must be provided since coreboot can't generate them itself:

 * Intel Flash Descriptor region
 * Intel Gigabit Ethernet firmware
 * Intel Management Engine

_First try: try not to use any binary blobs. The generated image won't be a complete image, however it should work after flashing, since ME and descriptor should stay as they are_

### 2.3. Check out and configure 

I followed the instructions on the coreboot documentation.

Pull the latest version of coreboot and update submodules:
```
eramon@caipirinha:~/dev/thinkpadw500$ git clone https://review.coreboot.org/coreboot
eramon@caipirinha:~/dev/thinkpadw500$ cd coreboot
eramon@caipirinha:~/dev/thinkpadw500$ git submodule update --init --checkout
```

The last step checks out a subrepository on the _3rdparty_ directory.

Generate config:
```
eramon@caipirinha:~/dev/thinkpadw500$ make menuconfig
```

 * On the _Mainboard_ menu, as _vendor_ select _Lenovo_
 * On the _Mainboard_ menu, as _model_ select _Thinkpad W500_
 * On the _Mainboard_ menu, as ROM chip size, select _4096 KB (4 MB))_
 * On the _Chipset_ menu, verify that _Unlock flash regions_ is selected
 * On the _Chipset_ menu, _Add Intel descriptor.bin file_: __NO__ 
 * On the _Chipset_ menu, _Add Intel ME/TXE Firmware_: __NO__ 
 * On the _Chipset_ menu, _Add gigabit ethernet configuration_: __NO__ 
 * On the _Payload_ menu, make sure that _SeaBIOS_ is selected

Save and exit. The configuration is saved to _.config_

The reason we chose NO for the descriptor, ME and GBE files is that we are NOT aiming to have a complete image but to have one image with coreboot only. The bits which are not covered by the coreboot built should not be overwritten on the flash chip and therefore stay as they are.


### 2.4. Build

_NOTE: I already had the build dependencies installed on my laptop_

Build the cross-compiler for all architectures:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ make crossgcc CPUS=4

Building toolchain using 4 thread(s).
Target architecture is i386-elf
```

Build cbfstool:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ cd util/cbfstool/
eramon@caipirinha:~/dev/thinkpadw500/coreboot/util/cbfstool$ make
```

CBFS stands for the Coreboot File System. 

_As far as Coreboot is concerned, the ROM image is a read-only file system. A special tool (cbfstool) can add extra components to a ROM image file. File deletion is not supported at all. The ROM image is composed (as a file on disk) by cbfstool and then the whole image is flashed into the actual flash chip._

Build ifdtool:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot/util/cbfstool$ cd ../ifdtool/
eramon@caipirinha:~/dev/thinkpadw500/coreboot/util/ifdtool$ make
```

idftool is a program to extract and dump Intel Firmware Descriptor information.


Build coreboot:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ make
```

At the end of the build process, following output is displayed:
```
Built lenovo/t400 (ThinkPad W500)

** WARNING **
coreboot has been built without an Intel Firmware Descriptor.
Never write a complete coreboot.rom without an IFD to your
board's flash chip! You can use flashrom's IFD or layout
parameters to flash only to the BIOS region.
```

This was expected. We did not override the flash descriptor bit on the chip, so the build process complains about the file descriptor missing on the image. 

After successfully build we had the image ready to be flashed:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ ls -la build/coreboot.rom 
-rw-r--r-- 1 eramon eramon 4194304 Dec 19 11:59 build/coreboot.rom
```

Since the descriptor is missing, inspect built rom with ifdtool does not work because the image does not have a flash descriptor.


## 5. Disassemble and connect

This part was the scariest one, at least for me. Even if I had a SPI flasher, I had never used it before. Reading the rom and flashing coreboot on the Librem is possible without having to disassemble the machine. And I had never bricked it (fingers crossed), so I have never needed the flasher.

However most chips don't allow this, so I had to disassemble to read the existing rom and to flash coreboot (the first time).

I found useful information about the wiring on the posts about installing libreboot on W500 (see links below). The chip has 16 pins, from them 5 which must be connected:

 * Pin number 2: 3.3 V
 * Pin number 7: CS#
 * Pin number 8: S0/SIO1
 * Pin number 15: S1/SIO0
 * Pin number 16: SCLK

_TODO: photo wiring_

## 6. Backup existing firmware

With the laptop connected to the chip, test the connection:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=512
```

Run flashrom to backup the existing firmware (bios):
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ sudo flashrom -c "MX25L6405D" -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -r firmware-orig.rom 
```

_Optional: use idftool to examine the original rom file_

## 7. Flash coreboot 

Flash coreboot:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ sudo flashrom -p linux_spi:dev=/dev/spidev0.0 -w build/coreboot.rom
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ sudo flashrom -p linux_spi:dev=/dev/spidev0.0 -w build/coreboot.rom
```

The second command should return VERIFIED - if so, coreboot has been successfully flashed to the device.

_NOTE: Once coreboot has been flashed initially, it can later be flashed internally, without neeed of the flashing device:_
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ sudo flashrom -p internal:laptop=force_I_want_a_brick -w coreboot.rom
```

Reboot and cross your fingers!

_TODO: Screenshot_

## 8. Ubuntu 

### 8.1. Re-partition

The Lenovo 500 had 150GB disk space, more than enough to keep the Windows installation - just in case, but specially because the laptop wasn't mine - and to install Ubuntu alongside it.

_TODO Repartition info_

### 8.2. Install

Boot from USB and install Ubuntu LTS on the new partition, following the installer instructions. 

_NOTE: How to install Ubuntu out of scope of this post._ 

_At the end: new, fast and up-to-date Ubuntu PC featuring open-source firmware and software :)_

_TODO: photo/screenshot_

## 9. Troubleshooting and Next Steps

_TODO: what is working and what not?_

## Links:

[Mainboards supported by coreboot](https://coreboot.org/status/board-status.html)

[Board:lenovo T400](https://www.coreboot.org/Board:lenovo/t400)

[Introducing CBFS](https://lennartb.home.xs4all.nl/coreboot/col5.html)

[Flashrom](https://dev.chromium.org/chromium-os/packages/cros-flashrom)

[Coreboot Build HowTo](https://www.coreboot.org/Build_HOWTO)

[Gentoo: Coreboot](https://wiki.gentoo.org/wiki/Coreboot)

[How to install libreboot on a ThinkPad W500](https://stafwag.github.io/blog/blog/2019/02/10/how-to-install-libreboot-on-a-thinkspad-w500/)

[Journey to freedom part 2](https://p1trson.blogspot.com/2017/01/journey-to-freedom-part-ii.html)
