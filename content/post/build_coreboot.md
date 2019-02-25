+++
math = true
date = "2019-02-18T10:00:00+02:00"
title = "Build coreboot"
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

/dev/coreboot-phil/3rdparty/blobs/mainboard/purism/librem_bdw

Get coreboot source code:
```
git clone https://review.coreboot.org/coreboot
```

Install build dependencies:
```
sudo apt-get install -y bison build-essential curl flex git gnat libncurses5-dev m4 zlib1g-dev flashrom
```

Install coreboot toolchain:
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

Clone the coreboot files from purism's github:
```
git clone https://source.puri.sm/coreboot/coreboot-files.git
```

Since coreboot was already on my laptop, I was able to extract the binary blobs from the machine itself. 

I analyzed purism's coreboot installation script (see references below) in order to find out how to get the binary blobs. I wrote a small script myself basing on theirs with only the command necessary to get the binary blobs:
```
#!/bin/bash -e
set -x

# CPU Microcode Blob URL
BDL_UCODE_URL="https://github.com/platomav/CPUMicrocodes/raw/18a85ffed180447aa16c2796146ff2698691eddf/Intel/cpu306D4_platC0_ver0000002A_2018-01-18_PRD_CC79BBDA.bin"

# Create blob directory
mkdir 3rdparty/blobs/mainboard/purism/librem_bdw

# 0. Get currently installed coreboot image
flashrom -p internal:laptop=force_I_want_a_brick,ich_spi_mode=hwseq -r coreboot-orig.rom

# 1. Copy me.bin from the output files of ifdtool (flash region 2)
util/ifdtool/ifdtool -x coreboot-orig.rom
cp flashregion_2_intel_me.bin 3rdparty/blobs/mainboard/purism/librem_bdw/me.bin

# 2. Get cpu_microcode_blob.bin from github
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

_NOTE: Since I relied on their script anyway, I did not check the hashes of the files, although I probably should._

## 2. Configure and build coreboot 

Copy the .config file from purism's coreboot-files. Run make menuconfig and save in order to update the configuration including all settings not included in the config. 

```
cp coreboot-files/configs/config.librem13v1 .config
make menuconfig

```

Following file paths must be corrected. I also replaced the bootsplash image with one of my own: 
```
-CONFIG_BOOTSPLASH_FILE="bootsplash.jpg"
+CONFIG_BOOTSPLASH_FILE="empanadilla.jpg"

-CONFIG_SEABIOS_BOOTORDER_FILE="bootorder-l13v1.txt"
+CONFIG_SEABIOS_BOOTORDER_FILE="coreboot-files/bootorder-l13v1.txt"
```

Now .config is ready and we can build coreboot:
```
make -j4
```

## 3. Install coreboot

Before flashing, reboot including the kernel parameter iomem=relaxed, otherwise flashrom will fail. For this, press "e" when the grub menu is shown and edit thekernel command line manually. 

After reboot, I flashed the new image I had just built:
```
sudo flashrom -p internal:laptop=force_I_want_a_brick -c MX25L6405D -w build/coreboot.rom 
```

Again a (cold) reboot - luckily the computer still booted! I was greeted by my own logo this time, after seeing on screen the SeeBios message.

The wifi wasn't working yet. However, that is another story and shall be told another time ;)

