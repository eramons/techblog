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

Getting a SPI 16 pin clip was going to take a couple of weeks, with Christmas vacations in between, so I had some time to prepare and do a little reading to better understand what it needed to be done and to learn a couple of facts about Coreboot and about firmware flashing. I could also start preparing the coreboot build.

### 1.2. ROM Size

To find out the ROM size, I booted the PC from a Ubuntu Live CD stick and ran the following:
```
ubuntu@ubuntu:~$ dmidecode | grep ROM\ Size
        ROM Size: 4096 kB
```

### 1.3. Propietary firmware 

Trying to learn something about the current firmware, I booted Ubuntu on the W500 with the live USB stick.

Install git and build dependencies:
```
sudo apt-get install -y bison build-essential curl flex git gnat libncurses5-dev m4 zlib1g-dev
```

Install flashrom:
```

```

_TODO: No flashrom package in ubuntu?_

## 2. Coreboot

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

_TODO Do I really need them or not?_

### 2.3. Check out and configure 

Follow the instructions on the coreboot documentation - adapting the configuration as necessary.

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
 * On the _Chipset_ menu, check _Add Intel descriptor.bin file_
 * On the _Chipset_ menu, verify that _Unlock flash regions_ is selected
 * _TODO_ On the _Chipset_ menu, _Add Intel descriptor.bin file_ YES or NO?
 * _TODO_ On the _Chipset_ menu, _Add Intel ME/TXE Firmware_ YES or NO?
 * _TODO_ On the _Chipset_ menu, _Add gigabit ethernet configuration_ YES or NO?

 * On the _Payload_ menu, make sure that _SeaBIOS_ is selected

Save and exit. The configuration is saved to _.config_

_TODO Some explanations about the configuration here_

### 2.3 Build

_NOTE: I already had the build dependencies installed on my laptop_

Build the cross-compiler for all architectures:
```
eramon@caipirinha:~/dev/thinkpadw500/coreboot$ make crossgcc CPUS=4

Building toolchain using 4 thread(s).
Target architecture is i386-elf
```

_TODO Does it make sense to build the cross-compiler like this? Why is the target architecture i386?_

_TODO Do I really need a cross-compiler? If the architecture is the same on my laptop?_

## 4. Disassemble and connect

This part was the scariest one, at least for me. Even if I had a SPI flasher, I had never used it before. My Librem allows read to and write from flash without having to disassemble the machine. I had never bricked my machine (fingers crossed), so I never needed it.

Apparently most chips don't allow this, so I had to disassemble. 

I found useful information about the wiring on the post about installing libreboot on W500. The chip has 16 pins, from them 5 which must be connected:

 * Pin number 2: 3.3 V
 * Pin number 7: CS#
 * Pin number 8: S0/SIO1
 * Pin number 15: S1/SIO0
 * Pin number 16: SCLK

## 5. Backup existing firmware

With my laptop connected to the chip, run flashrom to backup the existing firmware:
```

```

_TODO_

## 6. Flash coreboot 

_TODO_

## 5. Re-partition

_TODO Dual boot_

## 6. Install Ubuntu

Boot from USB and install Ubuntu LTS following the installer instructions. 

_NOTE: How to install Ubuntu out of scope of this post._ 

At the end we had an up-to-date Ubuntu PC at home, much faster as it was before, featuring open-source firmware and software :) 

_TODO: Foto_

## Links:

[Mainboards supported by coreboot](https://coreboot.org/status/board-status.html)

[Board:lenovo T400](https://www.coreboot.org/Board:lenovo/t400)

[Coreboot Build HowTo](https://www.coreboot.org/Build_HOWTO)

[How to install libreboot on a ThinkPad W500](https://stafwag.github.io/blog/blog/2019/02/10/how-to-install-libreboot-on-a-thinkspad-w500/)

[Journey to freedom part 2](https://p1trson.blogspot.com/2017/01/journey-to-freedom-part-ii.html)
