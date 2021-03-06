+++
math = true
date = "2021-05-31T09:00:00+02:00"
title = "WIP Troubleshooting Windows Boot with Coreboot/SeaBIOS"
tags = ["coreboot"]
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Boot Windows with Coreboot and SeaBIOS_

The refurbishing of the Thinkpad W500 was a success: the old laptop was working perfectly running open source firmware and software: Coreboot and Ubuntu. After building coreboot with seabios as payload, the laptop was able to boot ubuntu in bios mode. However, I kind of expected that seabios would be able to boot windows as well. 

_NOTE: I did not try to deactivate ME and didn't replace the GBe: for this I'm still using the proprietary blobs._ 

It was not the case: 
 * The existing Windows 7 installation which was still on the laptop won't boot (black screen)
 * A Windows 10 USB installation drive won't boot (black screen)
 * Neither _Safe mode_ or _Repair windows_ or _Start Windows normally_ would work 

I also tried to enable _boot logging_, by pressing F8 before booting and selecting the corresponding option in the menu. Afterwards I could find the file under _C:\Windows\ntbtlog.txt_, however I didn't see anything there which helped me out.

## 1. Resize ntfs partition  

At the beginning I was not really intending to keep windows, so I didn't bother to resize the windows partition before starting disassembling. However, after seeing windows wouldn't boot, I decided not to get rid of it but to just resize the partition to make room for installing linux. 

The best way to resize a windows partition is, well, to use windows. Since it was too late for that, I booted from an Ubuntu Live USB external drive and used _ntfsresize_ and _gparted_ to shrink the filesystem and the partition respectively.

Run _ntfsresize_ to resize the filesystem:
```
ubuntu@ubuntu:~$ sudo ntfsresize -s 150G /dev/sda2
ntfsresize v2017.3.23 (libntfs-3g)
Device name        : /dev/sda2
NTFS volume version: 3.1
Cluster size       : 4096 bytes
Current volume size: 319965622784 bytes (319966 MB)
Current device size: 319965626368 bytes (319966 MB)
New volume size    : 149999997440 bytes (150000 MB)
Checking filesystem consistency ...
100.00 percent completed
Accounting clusters ...
Space in use       : 100246 MB (31.3%)
Collecting resizing constraints ...
Needed relocations : 7695622 (31522 MB)
WARNING: Every sanity check passed and only the dangerous operations left.
Make sure that important data has been backed up! Power outage or computer
crash may result major data loss!
Are you sure you want to proceed (y/[n])? y
Schedule chkdsk for NTFS consistency check at Windows boot time ...
Resetting $LogFile ... (this might take a while)
Relocating needed data ...
100.00 percent completed
Updating $BadClust file ...
Updating $Bitmap file ...
Updating Boot record ...
Syncing device ...
Successfully resized NTFS> on device '/dev/sda2'.
You can go on to shrink the device for example with Linux fdisk.
IMPORTANT: When recreating the partition, make sure that you
  1)  create it at the same disk sector (use sector as the unit!)
  2)  create it with the same partition type (usually 7, HPFS/NTFS)
  3)  do not make it smaller than the new NTFS filesystem size
  4)  set the bootable flag for the partition if it existed before
Otherwise you won't be able to access NTFS or can't boot from the disk!
If you make a mistake and don't have a partition table backup then you
can recover the partition table by TestDisk or Parted's rescue mode.
```

Then use gparted to resize the NTFS partition to 150000 MiB (146.48 GiB):

![gparted](/techblog/img/gparted.png)

## 2. Install Ubuntu and fix Grub 

After resizing, install Ubuntu following the installer instructions. As for the partitioning, I created:

* A swap partition of 8GB (since the machine has 4GB RAM)
* A / partition of 100 GB (it remained 50 GB of free allocated space)

After completing the installation, I thought grub might be able to boot windows, but __the grub menu wasn't even showing__. 

The issue here was seabios not providing the graphical interface support expected by grub to show the boot menu. As a workaround - and to test if this was indeed the issue - I modified the grub configuration to show the menu as text only.

Modify _/etc/default/grub_ and uncomment/add following lines:
```
GRUB_TERMINAL=console
GRUB_GFXMODE=640x480
```

Update grub:
```
eramon@turicum:~/dev$ sudo update-grub
```

Reboot. Now the grub boot menu was there, and both ubuntu and windows were on the list. Grub wasn't able to boot windows either, though.

## 3. Get and compare ACPI tables

To troubleshoot the issue, compare the acpi tables from the vendor bios with the ones of coreboot. 

According to the coreboot documentation, many manufacturers do not provide complete acpi documentation, leading to incomplete acpi tables in coreboot.

### 3.1. Coreboot acpi tables

Install ubuntu acpi tools:
``` 
eramon@turicum:~$ sudo apt-get install acpica-tools
```

Copy ACPI tables from memory to disk:
```
eramon@turicum:~$ sudo acpidump > acpidata.dat
```

Extract them into raw, single tables:
```
eramon@turicum:~$ acpixtract -s FACP acpidata.dat

Intel ACPI Component Architecture
ACPI Binary Table Extraction Utility version 20180105
Copyright (c) 2000 - 2018 Intel Corporation

  FACP -     276 bytes written (0x00000114) - facp.dat
```

Disassemble/parse a specific table:
```
eramon@turicum:~$ iasl -d facp.dat
```

### 3.2. Go back to vendor bios

To compare the coreboot acpi tables with the vendor ones, we need to flash back the propietary bios. 

We need to be careful not to mess up the write protection. The best way is to flash only the bios region. For that, flashrom needs the flash descriptor file, which can be obtained using _ifdtool_.

Get files from last project:
```
eramon@turicum:~/dev$ git clone git@github.com:eramons/w500.git 
```

Extract flash descriptor layout from the original firmware image backup:
```
eramon@turicum:~/dev$ cd w500
eramon@turicum:~/dev/w500$ ifdtool -f w500.layout orig/w500_1.rom
File orig/w500_1.rom is 4194304 bytes
Wrote layout to w500.layout
``` 

Flash bios region only, using the layout file just created:
```
eramon@turicum:~/dev/w500$ sudo flashrom -c "MX25L3205D/MX25L3208D" -l w500.layout -i bios -w w500_1.rom -p internal
flashrom v1.2-312-g59e344e on Linux 5.4.0-73-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Using region: "bios".
Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
coreboot table found at 0x7d76e000.
Found chipset "Intel ICH9M".
Enabling flash write... OK.
Found Macronix flash chip "MX25L3205D/MX25L3208D" (4096 kB, SPI) mapped at physical address 0x00000000ffc00000.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```

Reboot. Original bios logo appears in screen. 

Boot into ubuntu. 

### 3.3. Vendor acpi tables

Repeat exactly the same steps already done in 3.1 to get the acpi tables.

### 3.4 Compare

_TODO What next? Learn to interprete and compare acpi tables_

## 4. VGA BIOS 

The fact that grub was working only in text mode pointed out that the same issue might be the one preventing windows to boot. 

### 4.1 Extract VGA BIOS binary

#### 4.1.1 Try bios_extract

Download and build _bios extract_:
```
eramon@turicum:~/dev$ git clone http://review.coreboot.org/p/bios_extract.git
eramon@turicum:~/dev$ cd bios_extract/
eramon@turicum:~/dev/bios_extract$ make
```

Try to extract vga bios using the _recommended method_:
```
eramon@turicum:~/dev/bios_extract$ ./bios_extract ../w500/orig/w500_1.rom
Using file "../w500/orig/w500_1.rom" (4096kB)
Error: Unable to detect BIOS Image type.
```

However, this did not work. I tried two alternative methods described on the coreboot documentation.

_NOTE: I assume the processor is not supported_

#### 4.1.2 Retrieval via Linux kernel

Use _lscpi_ to find out the PCI domain, bus, slot and function. 

```
root@turicum:/home/eramon/dev# lspci -tv
-[0000:00]-+-00.0  Intel Corporation Mobile 4 Series Chipset Memory Controller Hub
           +-01.0-[01]----00.0  Advanced Micro Devices, Inc. [AMD/ATI] RV635/M86 [Mobility Radeon HD 3650]
           +-02.0  Intel Corporation Mobile 4 Series Chipset Integrated Graphics Controller
           ...
```

Enable reading of the rom, copy the video blob an disable reading of the rom again:
```
root@turicum:/home/eramon/dev# echo 1 > /sys/devices/pci0000:00/0000:00:02.0/rom
root@turicum:/home/eramon/dev# cat /sys/devices/pci0000:00/0000:00:02.0/rom > vbios.dump
root@turicum:/home/eramon/dev# echo 0 > /sys/devices/pci0000:00/0000:00:02.0/rom
```

#### 4.1.3 Extraction from mapped memory

For this method, the machine must be booted with the vendor bios. 
```
root@turicum:/home/eramon/dev# cat /proc/iomem | grep 'Video ROM' | (read m; m=${m/ :*}; s=${m/-*}; e=${m/*-}; \
> dd if=/dev/mem of=vgabios.bin bs=1c skip=$[0x$s] count=$[$[0x$e]-$[0x$s]+1])
32768+0 records in
32768+0 records out
32768 bytes (33 kB, 32 KiB) copied, 0.24293 s, 135 kB/s
```

Install the _romheaders_ tool:
```
root@turicum:/home/eramon/dev# apt-get install fcode-utils
```

Use _romheaders_ to check the first binary:
```
root@turicum:/home/eramon/dev# romheaders vgabios.bin 

Image 1:
PCI Expansion ROM Header:
  Signature: 0x55aa (Ok)
  CPU unique data: 0x80 0xe9 0xff 0xf6 0x30 0x30 0x30 0x30
                   0x30 0x30 0x30 0x30 0x30 0x30 0x30 0x30
  Pointer to PCI Data Structure: 0x0040

PCI Data Structure:
  Signature: 0x50434952 'PCIR' (Ok)
  Vendor ID: 0x8086
  Device ID: 0x2a42
  Vital Product Data:  0x0000
  PCI Data Structure Length: 0x0018 (24 bytes)
  PCI Data Structure Revision: 0x00
  Class Code: 0x030000 (VGA Display controller)
  Image Length: 0x0080 blocks (65536 bytes)
  Revision Level of Code/Data: 0x0000
  Code Type: 0x00 (Intel x86)
  Last-Image Flag: 0x80 (last image in rom)
  Reserved: 0x0000

Platform specific data for x86 compliant option rom:
  Initialization Size: 0x80 (65536 bytes)
  Entry point for INIT function: 0xf705
```

The output of _romheaders_ looks good (it says _VGA Display controller_) and it is the same for both binaries, what I think is a good thing:
```
root@turicum:/home/eramon/dev# romheaders vgabios.bin > 1.txt
root@turicum:/home/eramon/dev# romheaders vbios.dump > 2.txt
root@turicum:/home/eramon/dev# diff 1.txt 2.txt 
root@turicum:/home/eramon/dev#
``` 

_NOTE: However, comparing the binaries using _sha1sum_ the binaries differ._

Following the order of recommended methods on the coreboot documentation: use the binary retrieved via the linux kernel. 

### 5. Re-build coreboot with VGA BIOS binary

Clone coreboot:
```
eramon@turicum:~/dev$ cd w500
eramon@turicum:~/dev/w500$ git clone https://review.coreboot.org/coreboot
eramon@turicum:~/dev/w500$ cd coreboot
eramon@turicum:~/dev/w500/coreboot$ 
```

Create directory for the binary blobs and copy them:
```
eramon@turicum:~/dev/w500/coreboot$ mkdir -p 3rdparty/blobs/mainboard/lenovo/t400
eramon@turicum:~/dev/w500/coreboot$ cp ../bin/flashregion_0_flashdescriptor.bin 3rdparty/blobs/mainboard/lenovo/t400/descriptor.bin
eramon@turicum:~/dev/w500/coreboot$ cp ../bin/flashregion_2_intel_me.bin 3rdparty/blobs/mainboard/lenovo/t400/me.bin
eramon@turicum:~/dev/w500/coreboot$ cp ../bin/flashregion_3_gbe.bin 3rdparty/blobs/mainboard/lenovo/t400/gbe.bin
```

Copy the config file:
```
eramon@turicum:~/dev/w500/coreboot$ cp ../config .config
```

Copy the VGA binary we generated before to the current directory and rename it: 
```
eramon@turicum:~/dev/w500/coreboot$ cp ../../vbios.dump ./vgabios.bin
```

Additional configuration for vga support:
```
eramon@turicum:~/dev/w500/coreboot$ make menuconfig
```

Go to the _Devices_ menu:
```
Devices -> Add a VGA BIOS image (vgabios.bin) VGA BIOS path and filename 
```

Exit and save.

Build:
```
eramon@turicum:~/dev/w500/coreboot$ make crossgcc CPUS=4
eramon@turicum:~/dev/w500/coreboot$ make
```

### 6. Flash coreboot

Theoretically, the coreboot image just built should work and might allow to boot windows. 

_NOTE: NOT TESTED YET_

Flash the image just built using the internal programmer.

## Links

[vga support](https://www.coreboot.org/VGA_support)

[dump video bios](https://01.org/linuxgraphics/documentation/development/how-dump-video-bios)

[ubuntu acpi](https://wiki.ubuntu.com/Kernel/Reference/ACPITricksAndTips)

[libgfxinit](https://doc.coreboot.org/gfx/libgfxinit.html)
