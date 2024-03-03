+++
math = true
date = "2024-02-19T10:00:00+02:00"
title = "Raspberry Pi Internet Radio"
tags = ["home automation", "raspberry pi"]
highlight = true

[header]
  caption = ""
+++

_Build an internet radio with an old Raspberry Pi 3 leveraging the Music Player Daemon (mpd)._

The ultimative goal is to refurbish a beautiful old radio from the 1930's. 

I intend to tell more about this radio in an upcoming post, but - spoiler alert - I can already tell that the original radio mechanism is probably beyond repair. The most feasible way seems to build a new device inside the old radio and bind the original buttons to the new platform, to control things like switching the power on and adjusting the sound volume.

At the time of this writing I don't have everything figured out yet. However, since all things have a beginning, I'll start with the part I'm more at ease at, that's the software side of things. My first goal will be to build an internet radio with an old Raspberry Pi 3 and a HiFiBerry amplifier, leveraging the Music Player Daemon.

_Music Player Daemon (MPD) is a flexible, powerful, server-side application for playing music. Through plugins and libraries it can play a variety of sound files while being controlled by its network protocol._

#### Table of contents:

 * [Choose the hardware](#1-hardware)
 * [Choose the operating system](#2-operating-system)
 * [Install and configure Arch Linux](#3-install-and-configure-archlinux-on-arm)
 * [Install Music Player Daemon](#4-install-music-player-daemon-on-archlinux)
 * [Try it out: MPC](#5-try-it-out-mpc)
 * [Understanding sound cards and devices](#6-understanding-sound-cards-and-sound-devices)
 * [HiFiBerry MiniAmp](#7-hifiberry-miniamp)
 * [Passive Speakers](#8-passive-speakers)
 * [Python MPC Client](#9-python-mpc-client-example)

### 1. Hardware

I wanted a small device with low power consumption. Since my intended use case - at least for now - only needs the mpd daemon, I didn't need much RAM, CPU or storage. I decided to give my old Raspberry Pi 3 a chance. My little contribution to the environment is to always try to re-use old hardware for my experiments :)

However, I still needed two little additional pieces of hardware for this project:

 * A HiFi Berry Amplifier: I went for the smallest HiFiBerry amplifier - the MiniAmp

 * I decided to get the "official" raspberry pi 3 power supply, since all micro USB chargers I had at home kept showing me an "under voltage" warning

### 2. Operating System 

Which operating system should I use? My preference has always been to use debian-based linux distributions, since I have been a debian user for years: in this case the Raspberry Pi OS should be logical option. However, the issue with it is that the mpd version in the corresponding debian package seems to be quite outdated. I tried building the application from source, including the manual build of some of the dependencies (ninja). This is too much hassle to mantain afterwards. 

_If the reader is interested in my notes about trying out this way:_
[Appendix B](#appendix-b)

Then I tried HifiberryOS, which is great and includes the mpd daemon and the mpc client, but it actually has much more functionality than I need. 

_If the reader is interested in my notes about trying out this way:_
[Appendix A](#appendix-a)

The third idea - which became the preferred option - was to install the arm flavour of the ArchLinux distribution on the Raspberry Pi 3 and then install the arch mpd package on it.

### 3. Install and configure ArchLinux on ARM

#### 3.1. Install 

Once again I'm amazed how good the Arch Linux documentation is. 

I just had to follow the instructions - see _links_ below - to install Arch Linux. I'm not repeating the instructions here, just summarizing a couple of things.

Use fdisk to partition the SDCard:

 * Delete all existing partitions, if there are any
 * Create a new primary partition of 200 MB of type W95 FAT32 (LBA)
 * Create a second primary partition of Linux type in the remaining space

It should look similar to this:
```
Device     Boot  Start      End  Sectors  Size Id Type
/dev/sdb1         2048   411647   409600  200M  c W95 FAT32 (LBA)
/dev/sdb2       411648 31116287 30704640 14.6G 83 Linux
```
Still following the instructions:

 * Create the FAT filesystem and mount it in a local "boot" directory
 * Create the ext4 filesystem and mount it in a local "root" directory
 * Download and extract the root filesystem into the root partition
 * Move the files under "/boot" on the root partition to the boot partition

Unmount and sync. 

#### 3.2. Configure

To do some initial configuration, I attached an external monitor via HDMI and an USB keyboard when booting for the first time.

_It's also possible to connect an ethernet cable and find out the IP address checking the router or access point leases. Then connect via SSH and configure the wifi via terminal. In that case, no monitor or keyboard are required._

Correct the time zone:
```
timedatectl set-timezone Europe/Zurich
```
_The date and time remained wrong after correcting the time zone. However, a while later, or maybe a reboot later, the date and time had auto-adjusted correctly._

To install the networkmanager I used the command _wifi-menu_ to set up wifi temporarily:
```
wifi-menu
```
Install _networkmanager_ and _sudo_ upgrading the pacman local database at the same time:
```
pacman -Syu networkmanager sudo
```
_Note: letting out the 'yu' would result in a 404 error._

Create a user and set the password:
```
useradd -m eramon
passwd eramon
```
Modify the sudoers file to allow the new user to use sudo.

Enable the networkmanager service, so the service will automatically start after reboot:
```
sudo systemctl enable NetworkManager.service
```
Start networkmanager:
```
systemctl start NetworkManager.service
```
Connect to the wifi network once manually:
```
nmcli device wifi connect myssid password mypwd
```
Find out the IP address, to connect over SSH later:
```
ip address
```
Reboot.

### 4. Install Music Player Daemon on ArchLinux

Connect to the raspi via SSH.

Again, I follow the official Arch Linux wiki instructions - see the _links_ below - to install the Music Player Daemon. Install the mpd package as root:
```
pacman -S mpd
```
Then enable the mpd service so it will run upon start:
```
systemctl enable mpd
```
Start the mpd daemon, for the first time, manually:
```
systemctl start mpd
```
According to the documentation, you can configure the mpd daemon system-wide or per user. Since the goal is to have the daemon started when the raspi boots, I don't necessarily to have a user-based configuration, I think, so I'm keeping things system-wide for now, even if this goes usually against best practices. 

The system-side configuration needs to be placed under _/etc/mpd/mpd.conf_

### 5. Try it out: MPC

To try out sending commands to mpd, I use the _mpc_ client. 

_mpc is a minimalist command line interface to MPD._

Install mpc:
```
pacman -S mpc
```
Mpd does not differentiate between songs with a beginning and an end, stored in a local or remote file, and a radio stream provided by a URL. When you add a URL to mpd, it's considered a track, and it's added to the playlist. Theoretically if having a playlist composed by different radio urls, going to the _next_ item in the playlist would emulate changing stations.

To play my favourite internet radio station I just needed to send following commands via the mpc client:
```
mpc add http://stream-uk1.radioparadise.com/mp3-128
mpc play
```
For the first test, I connected the HDMI output of the raspberry Pi to the HDMI input of our living-room home cinema system. The speakers of the system are pasive speakers, but the system has already an amplifier and a DAC (Digital Analog Converter) built on it. 

To test later directly directly with other passive speakers, I would need an analog audio output and an amplifier: the HiFiBerry Miniamp. At the moment of this first test it was already ordered but still on its way to me and was supposed to arrive in a couple of days. 

I would like the music to keep playing between reboots:

 * Since the mpd daemon service is enabled, it will start automatically
 * As it looks, the mpd playlist is persistent between reboots, so I don't have to add the URL manually every time
 * Every time at boot, I need to execute _mpc play_ so the music starts playing automatically 

For the last point, it was enough to add _mpc play_ at the end of _/etc/bash.bashrc_.

_NOTE: I need to review this point, I'm not quite positive that is the best way._

After reboot, radio paradise started playing out of the home cinema speakers :) 

For a bit later, instead of using mpc as the client, there are interesting possibilities: 

 * Use an android mpc client, to play music remotely 
 * In order to build custom physical controls, use the GPIO interface of the Raspberry Pi and translate the microcontroller commands into mpc calls, using python code and the corresponding python-mpd and _rpi.gpio_ libraries  

I'll tell more about the physical controls and the GPIO interface in the upcoming post, since it will be the basis for the refurbishing of the vintage radio I mentioned at the beginning.

### 6. Understanding sound cards and sound devices

To play around with the sound system and see the available sound cards and devices, install the _alsa-utils_:
```
sudo pacman -S alsa-utils
```
List available sound cards and playback devices:
```
[root@alarmpi eramon]# aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: vc4hdmi [vc4-hdmi], device 0: MAI PCM i2s-hifi-0 [MAI PCM i2s-hifi-0]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
```
List available output devices:
```
[root@alarmpi eramon]# aplay -L
null
    Discard all samples (playback) or generate zero samples (capture)
sysdefault
    Default Audio Device
sysdefault:CARD=vc4hdmi
    vc4-hdmi, MAI PCM i2s-hifi-0
    Default Audio Device
hdmi:CARD=vc4hdmi,DEV=0
    vc4-hdmi, MAI PCM i2s-hifi-0
    HDMI Audio Output
```
The sound card was listed correctly, the only device being the HDMI output the only device available so far. Later, when adding the HiFiBerry Miniamp, the sound output should have its own sound card listed.

_What I didn't find out: why is the audio jack not listed as a device? I didn't troubleshoot this further, since I wasn't going to use that output device anyway._

### 7. HiFiBerry MiniAmp

And then, on a sunny Thursday morning, my miniamp was delivered :D 

Connecting it to the raspberry pi was straighforward, just connect the small board over the GPIO pins. A thought arised: to use the GPIO interface later for switching on/off and controling the volume, I would need some kind of extension so I could still use the GPIO pins, which were now all blocked after connecting the miniamp. That should be possible, since only some of the pins are effectively used by miniamp.

I followed the instructions in the hifiberry documentation to make the raspi recognise the hifiberry sound card. It wasn't much, just modifying a couple of config files, as described next.

Apparently, only the hifiberry sound card must be enabled. To deactivate any other audio output, edit _/boot/config.txt_ changing the following line:
```
dtoverlay=vc4-kms-v3d,noaudio
```
Add the following line to the same file:
```
dtoverlay=hifiberry-dac
```
After that, the system recognises the new sound card and output device correctly:
```
[root@alarmpi eramon]# aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: sndrpihifiberry [snd_rpi_hifiberry_dac], device 0: HifiBerry DAC HiFi pcm5102a-hifi-0 [HifiBerry DAC HiFi pcm5102a-hifi-0]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
[root@alarmpi eramon]# aplay -L
null
    Discard all samples (playback) or generate zero samples (capture)
sysdefault:CARD=sndrpihifiberry
    snd_rpi_hifiberry_dac, HifiBerry DAC HiFi pcm5102a-hifi-0
    Default Audio Device
```
Modify /etc/mpd.conf to have mpd sending the sound to the right sound card:
```
# See: /usr/share/doc/mpd/mpdconf.example

pid_file "/run/mpd/mpd.pid"
db_file "/var/lib/mpd/mpd.db"
state_file "/var/lib/mpd/mpdstate"
playlist_directory "/var/lib/mpd/playlists"

# Audio Output
audio_output {
	type		"alsa"
        name            "sysdefault:CARD=sndrpihifiberry"
        mixer_type      "software"
}
```
Restart the service:
```
systemctl restart mpd
```

### 8. Passive speakers

And now? I needed some passive speakers to test if the little hifiberry amplifier was working. 

I found some in my cellar - which belonged to an old home sound system - so I connected two of them to the miniamp using their cables, following the polarity I found in the miniamp documentation (references below). Black is minus and red is plus, as far as I know:

![Connection](/techblog/img/vintage_radio/connection.jpg)

Surprisingly enough, it worked at the first try, the music was playing, much loudly as expected :D 

This is my first raspberry pi radio prototype:

![Miniamp](/techblog/img/vintage_radio/miniamp.jpg)

Now it's time to play a little around with the mpc client. First things first - adjust the volume before the neighbours come ringing at the door:
```
mpc volume 20
```
Then try to stop and play:
```
mpc stop
mpc play
```
Sounding good :) 

Being able to stop and play and controlling the volume via command line is the basis we'll need afterwards to do the same programatically.

### 9. Python mpc client example

Using the mpc client to try things out is a good start, however we want to see how this is possible programatically. 

_python-mpd2 is a Python library which provides a client interface for the Music Player Daemon._

Install the _python-pip_ package manager:
```
pacman -S python-pip
```
Install the _python-mpd2_ python library:
```
pip install python-mpd2
```
To try out the library, write a small script, which instanciates a new client, connects to the server, clear the playlist, add a internet radio URL, play the music, and print out some things.

The test script looks like this:
```
from mpd import MPDClient

client = MPDClient()
client.timeout = 10
client.idletimeout = None

# Connect to local server
client.connect("localhost", 6600)

# Print mpd version
print(" *** MPD Server version: " + client.mpd_version)

# Stop music, clear playlist, add URL, play
client.stop()
client.clear()
client.add("http://stream-uk1.radioparadise.com/mp3-128")
client.play()

# Set volume
client.volume(20)

# Print current song
print(" *** Current song:")
print(client.currentsong())

# Print client status
print(" *** Status:")
print(client.status())

# Print information of each song in the playlist
client.iterate = True
for song in client.playlistinfo():
    print(" *** Song: " + song["file"])

# Close and disconnect
client.close()
client.disconnect()
```
To try it out, execute:
```
python mpd-test.py
```
Later, when we have switches and buttons connecting to the GPIO interface, we'll translate the signals into mpd commands looking similar to the ones in the example above. Everything is prepared now for the next step, refurbishing an old vintage radio.

### Appendix A. 
___Alternative: HiFiBerry OS___

Even if it wasn't the approach I chose at the end, I'm including what I did when trying out HiFi Berry OS. It's a great software and I intend to try it again soon, it was just more than I needed for this project. 

Download HiFiBerryOS for the Raspberry Pi 3:
```
wget https://www.hifiberry.com/images/hifiberryos-pi3.zip
```
Unzip:
```
unzip hifiberryos-pi3.zip
```
Insert the sdcard in the laptop using an adapter. Find the right device name using dmesg - in my case it was sdb - then umount:
```
sudo umount /dev/sdb*
```
Copy the image to the sdcard:
```
dd if=hifiberryos-pi3.img of=/dev/sdb bs=4M
```
This take a couple of minutes. One finished, don't forget to sync:
```
sudo sync
```
To have SSH access to the raspberry pi later:
  * Mount the sdb1 partition in /mnt
  * Create a SSH file in /mnt
  * Exit, unmount and sync again

With that, the sdcard is ready with HiFiBerryOS for a 32-bit Raspberry Pi 3.

### Appendix B. 
___Discarded approach: build and install mpd on RaspberryPi OS___

When installing software, it's always worth of it to check the official available binaries. That's where I started when trying out this approach, however there were no available binaries for 32-bit ARM architectures. The only way would be to build mpd from source. I'm including the instructions to do so.

The precondition if choosing this approach would be that Raspberry OS is already up and running. Connect to the raspi via SSH and then install git:
```
sudo apt-get instal git
```
Clone the source from the official github repo:
```
git clone https://github.com/MusicPlayerDaemon/MPD.git
```
In the readme of the git repo, there is a link to the "basic installation instructions" under "read the manual". That looked like a good place to start. Extrapolating the list of dependencies I found there to the corresponding debian packages, I installed the following:
```
sudo apt-get install g++ meson ninja libboost-dev pkg-config
```
_According to the documentation, further dependencies will be needed later for the different MPD plugins, but I never got that far._

The packaged ninja version is too old for building mpd, so I had to build ninja manually as well, following the instructions in github. The available binaries were only for 64 bit architectures. Download the source code:
```
wget https://github.com/ninja-build/ninja/archive/refs/tags/v1.11.1.tar.gz
```
Unpack and go inside the ninja-1.11.1 directory. Running the following command should generate a ninja binary:
```
./configure.py --bootstrap
```
Delete the ninja package:
```
sudo apt-get remove ninja
```
Go back to the mpd source directory. Use the self-built ninja:
```
alias ninja="../ninja/ninja-1.11.1/ninja"
```
Build and install mpd:
```
meson . output/release --buildtype=debugoptimized -Db_ndebug=true
ninja -C output/release
sudo ninja -C output/release install
```
### Links and references

__Radio Paradise:__

https://radioparadise.com/

__Archlinux:__

https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3

https://wiki.archlinux.org/title/Help:Reading#Installation_of_packages

https://wiki.archlinux.org/title/System_time

https://wiki.archlinux.org/title/NetworkManager

https://wiki.archlinux.org/title/Music_Player_Daemon

__HiFiBerry amplifiers:__

https://www.hifiberry.com/amps

__MPD User's Manual:__

https://mpd.readthedocs.io/en/latest/user.html#

__MPC - Music Player Daemon:__

https://www.musicpd.org/clients/mpc/

__Python mpd2 library:__

https://python-mpd2.readthedocs.io/en/latest/topics/getting-started.html

https://python-mpd2.readthedocs.io/en/latest/topics/commands.html

__Miniamp - connect speakers:__

https://www.hifiberry.com/docs/data-sheets/datasheet-miniamp/attachment/miniamp-connection/

__Configure hifiberry sound card:__

https://www.hifiberry.com/docs/software/configuring-linux-3-18-x/

__Alternative operating systems / build your own mpd:__

https://www.hifiberry.com/hifiberryos/

https://www.raspberrypi.com/software/operating-systems/

https://github.com/MusicPlayerDaemon/MPD.git

https://mesonbuild.com/

https://ninja-build.org/

https://github.com/ninja-build/ninja

https://www.boost.org/

https://www.freedesktop.org/wiki/Software/pkg-config/



