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

1. Get needed files: source code, purism configuration and binary blobs 
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

I analyzed purism's coreboot installation script (see references below) in order to find out how to get the binary blobs. I wrote a small script myself basing on theirs with a minimal set of commands to get the six necessary blobs. See comments on the scripts for details. 
```
#!/bin/bash -e
# extract.sh - Extract binary blobs from librem 13 v1 already running coreboot
set -x

# CPU Microcode Blob URL (same link as Purism uses)
BDL_UCODE_URL="https://github.com/platomav/CPUMicrocodes/raw/18a85ffed180447aa16c2796146ff2698691eddf/Intel/cpu306D4_platC0_ver0000002A_2018-01-18_PRD_CC79BBDA.bin"

# Clone the coreboot files from purism's github:
git clone https://source.puri.sm/coreboot/coreboot-files.git

# Create blob directory, following coreboot file structure
mkdir 3rdparty/blobs/mainboard/purism/librem_bdw

# 0. Get the currently installed coreboot image with flashrom
flashrom -p internal:laptop=force_I_want_a_brick,ich_spi_mode=hwseq -r coreboot-orig.rom

# 1. Use ifdtool to extract the flash regions from the rom image 
# Copy me.bin which corresponds to the second flash region
util/ifdtool/ifdtool -x coreboot-orig.rom
cp flashregion_2_intel_me.bin 3rdparty/blobs/mainboard/purism/librem_bdw/me.bin

# 2. Download cpu_microcode_blob.bin
wget -O 3rdparty/blobs/mainboard/purism/librem_bdw/cpu_microcode_blob.bin $BDL_UCODE_URL

# 3. Extract mrc.bin from the rom image
util/cbfstool/cbfstool coreboot-orig.rom extract -n mrc.bin -f 3rdparty/blobs/mainboard/purism/librem_bdw/mrc.bin

# 4. Extract refcode.elf from the rom image
util/cbfstool/cbfstool coreboot-orig.rom extract -n fallback/refcode -f 3rdparty/blobs/mainboard/purism/librem_bdw/refcode.elf -m x86

# 5. Extract vgabios.bin from the rom image
util/cbfstool/cbfstool coreboot-orig.rom extract -n pci8086,1616.rom -f 3rdparty/blobs/mainboard/purism/librem_bdw/vgabios.bin

# 6. Get descriptor.bin from purism's repository (coreboot-files)
cp coreboot-files/descriptor-bdl.bin 3rdparty/blobs/mainboard/purism/librem_bdw/descriptor.bin
```

_NOTE: Since I relied on their script, files and sources,  I do not check the hashes on the script although it should be done._

## 2. Configure and build coreboot 

Copy the .config file from purism's coreboot-files. Run make menuconfig and save in order to update the configuration including all settings not included in the config. 

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

Before flashing, reboot including the kernel parameter iomem=relaxed, otherwise flashrom will fail. For this, press "e" when the grub menu is shown and edit the kernel command line manually. 

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
