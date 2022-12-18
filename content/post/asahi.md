+++
math = true
date = "2022-10-01T10:00:00+02:00"
title = "Asahi Linux kernel with u-root on Macbook"
tags = ["asahi"]
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Boot an Asahi Linux kernel and an u-root initramfs on a water-damaged Apple-Silicon Macbook_

At some point I encountered myself with a Macbook Air booting Asahi's m1n1 in proxy mode, without being able to boot MacOS or doing anything else with it, since the power button was fatally damaged.

## Introduction

__Why did I buy a Macbook and how I accidentally destroyed it__

Even if I definitely am a Linux-kind-of-person, I got myself an entry-level MacBook Air M1 which should fullfill the following purposes:
 * Act as a temporary work device until getting my work laptop from the company - since it was also going to be a Macbook 
 * Because it's light and handy, be the laptop of choice to take with me in vacation
 * The most important reason: to have an excuse to play around with Asahi Linux

_Asahi Linux is a project and community with the goal of porting linux to apple silicon macs._ 

I acquired this laptop at the end of March and middle April I had already got my company laptop: at least its main mission was fullfilled. It was middle May when I managed to empty some black tee all over the keyboard. Quick prognose: the keyboard and the touchpad had suffered an inminent death.

__Lessons learnt__
 * When you have water or any drink spilled over your laptop keyboard: shut it down immediately (even if you think _oh it's just a couple of drops_ )
 * If you want to incline the laptop to move the water away from the critical parts, do it so that the liquid goes to the back, since there is a slot there where it can come out

__And then what?__

The estimated cost of reparing was same as buying a new device, so not worth of it. My repair attempt consisting in disassembling and cleaning the burnt spots with ethanol, did not work, like at all. 
The good news: I had some fun ahead of me: I had installed the m1n1 proxy on the device before the fatal accident.

__Steps:__

_1. (BEFORE) On the mac:_
 - Partition the SSD and install m1n1

_2. On the linux laptop:_
 - Build m1n1
 - Build the Asahi Kernel
 - Build u-root

_3. (AFTER) On the linux laptop, conneced to the Mac via serial/USB:_
 - Boot the Asahi Kernel and u-root

## _BEFORE the damage_ 

As mentioned my Mac was mainly intented to be used to play around with Asahi Linux. Since Asahi is meant to be ran alongside MacOS and because macbooks are almost impossible to brick, there was no risk in installing m1n1 even if I still needed to use MacOS for work. 

_m1n1 is the bootloader developed by the Asahi Linux project to bridge the Apple (XNU) boot ecosystem to the Linux boot ecosystem._

### 1. On the Mac 

#### 1.1 Partition the SSD and install m1n1

Following the instructions on the wiki, I prepared the partitions and install m1n1:

_TODO Find my notes to the partitioning_ 

## _AFTER the damage_

After the incident, the boot menu did not appear anymore when hold-pressing the power button, apparently it was also damaged. When booting the system the Asahi logo would greet me, followed for the m1n1 in proxy mode. I couldn't get anywhere else but there.

At this point, what was intending to achieve? Even if it wasn't that clear to me how, the idea I got in mind was the following:

 * As I understood, booting Linux in a Silicon Mac works as follows: _m1n1 -> bootloader -> kernel -> rootfs_
 * Bootloader: the Asahi project uses u-boot as the bootloader, here where I intended to do things a little differently: to follow Linuxboot approach using a (reduced) Asahi kernel with u-root as initramfs
 * Kernel: Asahi (relying in the .config of the ARCH distribution)
 * Rootfs: Debian

### 2. On the linux laptop

#### 2.1. Build m1n1

m1n1 is not just the bootloader installed on the Mac, is also a binary used on a host computer to send commands to the Mac when it's in Asahi Proxy Mode.

_The m1n1 proxy mode provides a USB device interface for booting a linux kernel directly from the host computer._

Clone and build m1n1:
```
eramon@caipirinha:~/dev$ git clone --recursive https://github.com/AsahiLinux/m1n1.git
eramon@caipirinha:~/dev$ cd m1n1
eramon@caipirinha:~/dev/m1n1$ make
```

#### 2.2. Build asahi kernel

Install the cross-compiler:
```
eramon@caipirinha:~/dev/asahi/m1n1$ sudo apt-get install make gcc-aarch64-linux-gnu
```
Clone the asahi linux kernel:
```
eramon@caipirinha:~/dev$ git clone https://github.com/AsahiLinux/linux.git
```
I relied in the files of the ARCH Asahi Linux distribution:
 * To know which version of the asahi linux kernel to build
 * To have a working kernel config 

_NOTE: I asked for a recommendation regarding which .config to use in the Asahi IRC channel_ 

_NOTE: even if this config is the recommended one for the Asahi Linux distribution, it might be not the ideal choice to use it as a bootloader, following the linuxboot approach. The necessary modules should be included directly in the kernel._

To know which source code tag to build, I took a look inside the file https://github.com/AsahiLinux/PKGBUILDs/blob/main/linux-asahi/PKGBUILD:
```
_rcver=5.19
_asahirel=1
``` 
Checkout the source corresponding to 5.19-1:
```
eramon@caipirinha:~/dev$ cd linux/
eramon@caipirinha:~/dev/linux$ git fetch --tags --all
eramon@caipirinha:~/dev/linux$ git checkout asahi-5.19-1
```
Get the config file from the ARCH distribution:
```
eramon@caipirinha:~/dev/linux$ wget https://raw.githubusercontent.com/AsahiLinux/PKGBUILDs/main/linux-asahi/config
eramon@caipirinha:~/dev/linux$ cp config .config
```
Configure and build the kernel:
```
eramon@caipirinha:~/dev/linux$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make olddefconfig
eramon@caipirinha:~/dev/linux$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make Image.gz dtbs
```

After built, following files have been generated:
 * The compressed kernel image: Image.gz
 * The MacBook Air device tree, knowing that the product name is J313AP and the SoC is T8103 the product name is J313AP and the SoC is T8103

Copy the above mentioned files, which we're going to need later, close to the m1n1 binary: 
```
eramon@caipirinha:~/dev/linux$ mkdir ../m1n1/files
eramon@caipirinha:~/dev/linux$ cp arch/arm64/boot/dts/apple/t8103-j313.dtb ../m1n1/files/
eramon@caipirinha:~/dev/linux$ cp arch/arm64/boot/Image.gz ../m1n1/files/
```

#### 2.3. u-root

u-root is one of the deliverables of the Linuxboot project. 

Linuxboot replaces specific firmware functionality with a Linux kernel and runtime. 

One of the outcomes of linuxboot is u-root: a minimal initrams written in Go.

We are using u-root to build an initramfs to be started after the asahi kernel boots.

Clone the u-root source code:
```
eramon@caipirinha:~/dev$ git clone https://github.com/u-root/u-root
eramon@caipirinha:~/dev$ cd u-root 
```
Install go:
```
eramon@caipirinha:~/dev/u-root$ wget https://go.dev/dl/go1.19.linux-amd64.tar.gz
eramon@caipirinha:~/dev/u-root$ sudo rm -rf /usr/local/go
eramon@caipirinha:~/dev/u-root$ sudo tar -C /usr/local -xzf go1.19.linux-amd64.tar.gz
```
Build and install u-root:
```
eramon@caipirinha:~/dev/u-root$ export PATH=$PATH:/usr/local/go/bin
eramon@caipirinha:~/dev/u-root$ export GOPATH=/home/eramon/dev/go
eramon@caipirinha:~/dev/u-root$ go install github.com/u-root/u-root
```
The u-root binary will be installed under $GOPATH:
```
eramon@caipirinha:~/dev/u-root$ export PATH=$PATH:/home/eramon/dev/go/bin
```
Run u-root with the cross-architecture prefix:
```
eramon@caipirinha:~/dev/u-root$ GOARCH=arm64 u-root
```
An initramfs for the arch64 architecture was generated:
```
09:59:10 Successfully built "/tmp/initramfs.linux_arm64.cpio" (size 11877784)
```
Same as before, copy the file close to the m1n1 binary:
```
eramon@caipirinha:~/dev/u-root$ cp /tmp/initramfs.linux_arm64.cpio ../m1n1/files/
```
### 3. Boot the kernel with u-root

Pre-condition: you need to have Python3 installed on the system

Opening the lid of the macbook air is all the preparation we need, since:
 * Macbooks boot automatically when opening the lid
 * The Options menu does not work anymore, the laptop boots straight to the asahi proxy

Connect the Macbook with an USB-C cable to the linux laptop, where m1n1, the linux kernel and u-root were built. 

Go to the directory where m1n1 was built:
```
eramon@caipirinha:~$ cd dev/m1n1/
```
There are two different ways to boot the kernel and the initramfs with m1n1:
 * Directly: booting a linux kernel directly, passing a gzipped kernel, the device tree and optionally an initramfs
 * Virtualized: under the m1n1 hypervisor. M1n1 will first load as a guest inside m1n1 - what the asahi developers call inception - and then load the embedded kernel and initramfs

The first way do not - at the time of this writting - work properly. The reason is that the console is not initialised properly, so even if it's indeed working, we do not see anything else than the asahi logo frozen on the screen.

For the sake of having everything documented, that's how to boot using the first way:
```
eramon@caipirinha:~/dev/asahi/m1n1$ export M1N1DEVICE=/dev/ttyACM0
eramon@caipirinha:~/dev/asahi/m1n1$ python3 proxyclient/tools/linux.py -b 'earlycon debug rootwait' files/Image.gz files/t8103-j313.dtb files/initramfs.linux_arm64.cpio.gz
```
m1n1 communicates with the host via serial, which in the host is /dev/ttyACM1. I used picocom to see the output there, but it always crashed at some point. 

The second way allows to do the same while having a serial console, so we have the output and keyboard input through the host.

First, gzip the initramfs, since it must be gzip compressed:
```
eramon@caipirinha:~/dev/m1n1$ gzip -9 files/initramfs.linux_arm64.cpio
```
Build the binary concatenating the m1n1 binary - the older format .macho - with the DTB, the initramfs and the kernel image:
```
eramon@caipirinha:~/dev/m1n1$ cat build/m1n1.macho <(echo 'chosen.bootargs=earlycon debug rw') files/t8103-j313.dtb files/initramfs.linux_arm64.cpio.gz files/Image.gz > files/m1n1-linux-uroot.macho
```
In another terminal, prepare the serial console, opening the second port with a serial terminal:
```
eramon@caipirinha:~/dev/m1n1$ sudo apt-get install picocom
eramon@caipirinha:~/dev/m1n1$ picocom --omap crlf --imap lfcrlf -b 500000 /dev/ttyACM1
```
You should see the following:
```
Type [C-a] [C-h] to see available commands

Terminal ready
```
Go back to the first terminal and run the binary as guest to the m1n1 hypervisor:
```
eramon@caipirinha:~/dev/m1n1$ python3 proxyclient/tools/run_guest.py files/m1n1-linux-uroot.macho
```
If everything goes well, we should be greeted by the u-root logo in the serial console:

```
1970/01/01 00:00:00 Welcome to u-root!
                              _
   _   _      _ __ ___   ___ | |_
  | | | |____| '__/ _ \ / _ \| __|
  | |_| |____| | | (_) | (_) | |_
   \__,_|    |_|  \___/ \___/ \__|

/# 
```
With u-root we get a shell with a minimalistic set of commands. 

What next? From there, boot a linux kernel and mount a root system. That's a whole story on its own, so enough material for another blog post :)

The first outcome was that the disk devices were not visible from the u-root environment, probably because the nvme controller was compiled as a module in the kernel configuration I had chosen.


### Links:

https://github.com/AsahiLinux/m1n1
https://github.com/AsahiLinux/docs/wiki/m1n1%3AUser-Guide
https://github.com/AsahiLinux/PKGBUILDs/tree/main/linux-asahi
https://github.com/AsahiLinux/linux
https://github.com/AsahiLinux/docs/wiki/Devices
https://go.dev/doc/install
https://www.linuxboot.org/
https://github.com/linuxboot/book
https://github.com/u-root/u-root
