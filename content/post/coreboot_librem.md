+++
math = true
date = "2018-12-04T10:00:00+02:00"
title = "Install coreboot on the Librem 13 v.1"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

*Blah*

Blah:

Goal:

* Replace the bootloader shipped with the Librem 13 v.1 through coreboot




[https://puri.sm/coreboot](https://puri.sm/coreboot)

Some of the information on the site is still usefull, however the script provided with the instructions is old and unmantained. 


## 1. Migrate to a non-UEFI installation 

The librem supports Coreboot with SeaBios as payload. Tianocore is not supported, so an UEFI installation won't boot after replacing the bootloader.

Tasks:

* Check if the installed debian system is a UEFi or a BIOS install
* Check if the partition layout is MBR or GPT
* Resize the existing partitions in order to free 100 MB at the beginning of the disk 
* Create a new 100 MB partition 
* Re-install  

When I got my librem 13 on September 2016, I replaced the shipped OS (PureOS) through an UEFI-based debian system. In order to make sure the OS is indeed installed on UEFI mode, I quickly checked:

- There is a /sys/firmware/efi on the rootfs
- There is a fat32 partition (called "efi") mounted on /boot/efi

There is two existing partition schemes:  
 
* MBR (Master Boot Record): older
* GPT (GUID Partition Table): newer - required to boot operating systems in UEFI mode

Since the OS installation is an UEFI mode debian, it's sure that the disk scheme is GPT. To check this anyway, check the output of fdisk:
```
sudo fdisk /dev/sda
...
Disklabel type: gpt
```

Or even better gdisk:
```
sudo gdisk /dev/sda
...
Partition table scan:
	MBR: protective
	BSD: not present
	APM: not present
	GPT: present
```

I use grub as 2nd stage bootloader. In non-EFI mode, grub is able to boot from a partitioned GPT disk if using a compatible boot scheme consisting on having a special 1 MB partition at the start of the disk. 

I checked the partitions on the librem (you can use either gparted or gdisk):
	
1. /dev/sda1 - boot:  ext4 partition mounted on /boot (?) with size=500 MB (113 MB used)   
2. /dev/sda2 - efi: fat32 partition mounted on /boot/efi with size=500 MB and flags "boot" and "esp" set
3. /dev/sda3 - system: crypt-luks (encrypted) partition

The boot partition must be resized and moved in order to have a new 100 MB partition at the beginning of the disk. Since the partitions must be unmounted, I created a live debian stretch usb drive. 

(https://www.debian.org/CD/live)[https://www.debian.org/CD/live]

```
sudo dd if=/home/eramon/Downloads/debian-live-9.8.0-amd64-gnome.iso of=/dev/sdb bs=1M
```

_Reboot to the live system on the usb._

Boot from the USB drive (pressing F2 to enter setup and selecting boot override). 

On the live CD, I used gparted to resize the boot partition, freing 100 MB at the beginning of the disk. 
```
sudo gparted
...

_Reboot to the debian system._

I used gdisk to create the new 100 MB partition for grub:

Command (? for help): n
Partition number (4-128, default 4):
First sector (34-976773134, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-206847, default = 206847) or {+-}size{KMGTP}: +100M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): ef02
Changed type of partition to 'BIOS boot partition'

Command (? for help): p 
...
Number  Start (sector)    End (sector)  Size       Code  Name
   1          206848         1023999   399.0 MiB   8300  boot
   2         1024000         2047999   500.0 MiB   EF00  efi
   3         2048000       976771071   464.8 GiB   8300
   4            2048          206847   100.0 MiB   EF02  BIOS boot partition

Command (? for help): wq
```

Modify /etc/fstab and comment out the mount of the /boot/efi partition:
```
# Removed on 2019 February 18th to switch to a non-EFI system to be able to install coreboot:
# UUID=559D-9E15  /boot/efi       vfat    umask=0077      0       1
```

Finally, I removed grub-efi and installed grub2:
```
sudo dpkg --purge grub-efi
sudo apt-get -f autoremove
sudo apt-get update
sudo apt-get grub2
sudo grub install --target=i386-pc /dev/sda
``` 

Some notes:
- The target parameter must be specified, otherwise grub tries to install grub on efi mode. 
- Grub must be installed to the disk (/dev/sda) an not to the partition (/dev/sda2 - e.g. the boot partition) as it would be for grub-efi.

Reboot to see if it worked. To be sure, I went into the bios and tried a boot override. There were two different boot options, both for the same disk. One of them is UEFI boot and the other is the non-UEFI one. Both worked.

An useful link:
(ArchLinux Grub Documentation)[https://wiki.archlinux.org/index.php/GRUB#GUID_Partition_Table_.28GPT.29_specific_instructions]

## 2. Install Coreboot

As mentioned, the script provided on the purism website didn't work anymore. 

support@puri.sm A

```
sudo apt-get install libftdi1 libftdi1-dev

```

```
Enabling flash write... Error accessing ICH RCRB, 0x4000 bytes at 0x00000000fed1c000
/dev/mem mmap failed: Operation not permitted
```

According to the flashrom FAQ, it is possible to avoid this error if overriding the kernel command line to include jjkkkkkk iomem=relaxed 
iomem=relaxed 

```
eramon@caipirinha:~/dev/coreboot-l13v1-oem$ sudo ./flashrom -p internal:laptop=force_I_want_a_brick -w coreboot-l13v1.rom
flashrom v1.0 on Linux 4.19.0-2-amd64 (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
========================================================================
WARNING! You seem to be running flashrom on an unsupported laptop.
Laptops, notebooks and netbooks are difficult to support and we
recommend to use the vendor flashing utility. The embedded controller
(EC) in these machines often interacts badly with flashing.
See the manpage and https://flashrom.org/Laptops for details.

If flash is shared with the EC, erase is guaranteed to brick your laptop
and write may brick your laptop.
Read and probe may irritate your EC and cause fan failure, backlight
failure and sudden poweroff.
You have been warned.
========================================================================
Proceeding anyway because user forced us to.
Found chipset "Intel Broadwell U Premium".
This chipset is marked as untested. If you are using an up-to-date version
of flashrom *and* were (not) able to successfully update your firmware with it,
then please email a report to flashrom@flashrom.org including a verbose (-V) log.
Thank you!
Enabling flash write... Warning: SPI Configuration Lockdown activated.
Enabling hardware sequencing because some important opcode is locked.
OK.
Found Programmer flash chip "Opaque flash chip" (8192 kB, Programmer-specific) mapped at physical address 0x0000000000000000.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```
