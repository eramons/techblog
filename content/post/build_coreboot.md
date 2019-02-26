+++
math = true
date = "2019-02-25T10:00:00+02:00"
title = "Build Coreboot"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

__DISCLAIMER: try anything described here at your own risk! These are just my notes. That it did work for me then does not mean this would work for you now.__

After successfully installing coreboot on my librem 13 v1, I noticed that unfortunately the wifi wasn't working anymore. Since the coreboot version I flashed previously wasn't quite new, I decided to build a new rom image myself using the newest coreboot source code.

__Goal:__

_Replace coreboot with an up-to-date, self-built coreboot version_

Tasks:

1. Get required files: coreboot source code, purism config and librem binary blobs 
2. Configure and build coreboot 
3. Install coreboot 


## 1. Get the files

Get coreboot source code:
```
git clone https://review.coreboot.org/coreboot
```

Install build dependencies:
```
sudo apt-get install -y bison build-essential curl flex git gnat libncurses5-dev m4 zlib1g-dev
```

Install flashrom:
```
sudo apt-get install flashrom
```

Build coreboot toolchain:
```
cd coreboot
make crossgcc-i386 CPUS=4
```

Build cbfstool and ifdtool:
```
cd util/cbfstool
make
cd ../util/ifdtool
make
cd ..
```

Since coreboot was already on my laptop, I was able to extract the binary blobs from the machine itself. 

I analyzed purism's coreboot installation script (see references below) in order to find out how to get the binary blobs. 

Create blob directory, following coreboot file structure:
```
mkdir 3rdparty/blobs/mainboard/purism/librem_bdw
```

Get the currently installed coreboot image with _flashrom_:
```
flashrom -p internal:laptop=force_I_want_a_brick,ich_spi_mode=hwseq -r coreboot-orig.rom
```

Use _ifdtool_ to extract the flash regions from the rom image:
```
sudo ./util/ifdtool/ifdtool -x coreboot-orig.rom 
```

Output of _idftool_:
```
File coreboot-orig.rom is 8388608 bytes
  Flash Region 0 (Flash Descriptor): 00000000 - 00000fff 
  Flash Region 1 (BIOS): 00200000 - 007fffff 
  Flash Region 2 (Intel ME): 00001000 - 001fffff 
  Flash Region 3 (GbE): 00fff000 - 00000fff (unused)
  Flash Region 4 (Platform Data): 00fff000 - 00000fff (unused)
```

Copy the file containing the second flash region to _me.bin_:
```
cp flashregion_2_intel_me.bin 3rdparty/blobs/mainboard/purism/librem_bdw/me.bin
```

Download _cpu\_microcode\_blob.bin_:
```
wget -O 3rdparty/blobs/mainboard/purism/librem_bdw/cpu_microcode_blob.bin "https://github.com/platomav/CPUMicrocodes/raw/18a85ffed180447aa16c2796146ff2698691eddf/Intel/cpu306D4_platC0_ver0000002A_2018-01-18_PRD_CC79BBDA.bin" 
```

Extract _mrc.bin_ from the rom image:
```
util/cbfstool/cbfstool coreboot-orig.rom extract -n mrc.bin -f 3rdparty/blobs/mainboard/purism/librem_bdw/mrc.bin
```

Extract _refcode.elf_ from the rom image:
```
util/cbfstool/cbfstool coreboot-orig.rom extract -n fallback/refcode -f 3rdparty/blobs/mainboard/purism/librem_bdw/refcode.elf -m x86
```

Extract _vgabios.bin_ from the rom image:
```
util/cbfstool/cbfstool coreboot-orig.rom extract -n pci8086,1616.rom -f 3rdparty/blobs/mainboard/purism/librem_bdw/vgabios.bin
```

Clone the _coreboot-files_ from purism's github repository to get _descriptor.bin_:
```
git clone https://source.puri.sm/coreboot/coreboot-files.git
cp coreboot-files/descriptor-bdl.bin 3rdparty/blobs/mainboard/purism/librem_bdw/descriptor.bin
```

I'm including the output of _sha256sum_ on all six binaries - which I compared with the ones provided by purism on their script as a security check:
```
69537c27d152ada7dce9e35bfa16e3cede81a18428d1011bd3c33ecae7afb467  3rdparty/blobs/mainboard/purism/librem_bdw/cpu_microcode_blob.bin
be34b19b4de387a07d4fc859d2e4ee44723756f5f54552f236136679b4e52c46  3rdparty/blobs/mainboard/purism/librem_bdw/descriptor.bin
1e8f08c3eb31a0fdb91ec0222d4398b9192141502941a5262e9155915ffb6991  3rdparty/blobs/mainboard/purism/librem_bdw/me.bin
dd05ab481e1fe0ce20ade164cf3dbef3c479592801470e6e79faa17624751343  3rdparty/blobs/mainboard/purism/librem_bdw/mrc.bin
8a919ffece61ba21664b1028b0ebbfabcd727d90c1ae2f72b48152b8774323a4  3rdparty/blobs/mainboard/purism/librem_bdw/refcode.elf
e1cd1b4f2bd21e036145856e2d092eb47c27cdb4b717c3b182a18d8c0b1d0f01  3rdparty/blobs/mainboard/purism/librem_bdw/vgabios.bin
``` 

## 2. Configure and build coreboot 

Copy the _.config_ file from purism's _coreboot-files_ downloaded before. Then run _make menuconfig_ and select _save_ in order to update the configuration including all settings not yet included:

```
cp coreboot-files/configs/config.librem13v1 .config
make menuconfig
```

Following file paths must be corrected: 
```
-CONFIG_BOOTSPLASH_FILE="bootsplash.jpg"
+CONFIG_BOOTSPLASH_FILE="coreboot-files/bootsplash.jpg"

-CONFIG_SEABIOS_BOOTORDER_FILE="bootorder.txt"
+CONFIG_SEABIOS_BOOTORDER_FILE="coreboot-files/bootorder-l13v1.txt"
```

Now .config is ready and we can build coreboot:
```
make -j4
```

## 3. Install coreboot

Before flashing, reboot including the kernel parameter _iomem=relaxed_, otherwise flashrom will fail. For this, press "e" when the grub menu is shown and edit the kernel command line manually. 

After reboot, I flashed the new image I had just built:
```
sudo flashrom -p internal:laptop=force_I_want_a_brick -c MX25L6405D -w build/coreboot.rom 
```

And then a cool reboot - luckily the computer still booted :D 

I saw first the SeeBios message, then the Purism Logo and then the Grub boot menu.   

The wifi wasn't working yet. However, that is another story and shall be told another time ;)

References and sources:

[Coreboot Rookie Guide](https://doc.coreboot.org/lessons/lesson1.html)

[https://puri.sm/coreboot](https://puri.sm/coreboot)

[Flashrom FAQ](https://www.flashrom.org/FAQ)

[Coreboot on Librem](http://localhost:1313/techblog/post/coreboot_librem)
