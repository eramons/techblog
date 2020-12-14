+++
math = true
date = "2020-12-09T15:00:00+02:00"
title = "WIP /e self-hosting and /e OS on LG G3"
tags = []
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Rely on the /e platform for private cloud self-hosting and as android operating system_

/e is a mobile ecosystem that:

 * is open source
 * is presented as an environment which values privacy
 * is un-googled but still compatible with android apps

The /e/ ROM is a fork of Android (more precisely from LineageOS).

The /e/ cloud includes several applications and it’s built upon NextCloud, Postfix, Dovecot and OnlyOffice. It’s been integrated to offer a single login identity in /e/OS as well as online, where users can retrieve their data from the ecloud global services. It's also possible to self-host a private cloud on an own server.

## Pre-conditions

For this project, I needed a client and a server. Luckily my old LG G3 (year 2014) - which still worked perfectly - is supported by /e (last build was from 2020/10/01 - just two months old at the time of this writting). So as it looked, it was still maintained. I had rooted the device, installed TWRP and installed LineageOS on it a couple of years back.

As a server I wasn't so sure. I would have liked to host /e on my K3s cluster on the Raspberry Pi 4, but unfortunately arm was not supported at the time, only x86 or x86-64. For a first experimental setup I decided to host the /e software on a vm directly on my laptop.

_Thinking about purchasing hardware to host the software later, I would try to find a small server with little power consumption, if possible not overpriced and with coreboot support (since this could be handy for later projects ;))_

 * Client side: install /e operating system on LG G3 d855 (International) 
 * Server side: install /e self-hosting on a VM

For the self-hosting, I also needed:
 
 * A hosted public domain (_mydomain.com_)
 * A DNS provider to register the necessary DNS entries (_Cloudflare_)

Assuming all the pre-conditions were met, following milestones had to be reached:

 1. Install /e on the mobile phone
 2. Set up Vagrant as virtualization environment for the server
 3. Set up DNS on the home network
 4. Install /e self-hosting on the VM 

## 1. Install /e on LG G3

Well, the installation was pretty straightforward, since I already had some things in place:

 * The device was rooted
 * The device had a custom recovery installed - TWRP
 * The device had adb debugging enabled
 * I had adb installed on my laptop 

First thing was to connect the device to my laptop. Then reboot to recovery through adb:
```
eramon@caipirinha:~/dev/e$ adb reboot recovery
```

I followed the instructions online to wipe the device from the TWRP recovery, completely wiping everything, including the swap and the cache. 

Then I downloaded the image from the dowloads website and installed the image using sideload:
```
eramon@caipirinha:~/dev/e$ adb sideload e-0.12-n-2020100176273-dev-d855.zip 
Total xfer: 1.00x      
```

After restart the new operating system was running on the phone.

Looking nice :)
![LGG3](/techblog/img/lgg3_e.jpg)

## 2. Set up Vagrant

Before having a server to install the /e self-hosting software on, I decided to try it out running it locally on a virtual machine. As virtualization environment I would use _Vagrant_, since I have been wanting to try it out for a while. 

Vagrant is a tool for building and managing virtual machine environments in a single workflow. 

First of all, install linux-headers and virtualbox:
```
eramon@caipirinha:~/dev/vagrant$ sudo apt-get install linux-headers-amd64
eramon@caipirinha:~/dev/vagrant$ sudo apt-get install virtualbox
```

I downloaded Vagrant ("linux" version) and installed it manually. Using the available debian package installs Vagrant with libvirt, and I wanted to use Vagrant with Virtualbox. 


```
eramon@caipirinha:~/dev/vagrant$ wget https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_linux_amd64.zip

eramon@caipirinha:~/dev/vagrant$ unzip vagrant_2.2.7_linux_amd64.zip
Archive:  vagrant_2.2.7_linux_amd64.zip
  inflating: vagrant
```

_NOTE: The .deb file from the Vagrant downloads page gave me an error when trying to install it with dpkg._

Move binary to bin folder:
```
eramon@caipirinha:~/dev/vagrant$ sudo mv vagrant /usr/local/bin/
```

Pull and run an Ubuntu Bionic VM:
```
eramon@caipirinha:~/dev/vagrant$ vagrant init ubuntu/bionic64
```

When trying to connect, I got an error:
```
==> default: Rsyncing folder: /home/eramon/dev/vagrant/ => /vagrant
There was an error when attempting to rsync a synced folder.
Please inspect the error message below for more info.
```

Googling around I found out this is a known issue and that there is a workaround:
```
eramon@caipirinha:~/dev/vagrant$ export OPENSSL_CONF=/etc/ssl/
```

After that, I could connect to the new VM:
```
eramon@caipirinha:~/dev/vagrant$ vagrant ssh
```

## 3. DNS

For self-hosting /e you need to have at least a domain registered. I already own _mydomain.com_ and I use Cloudflare for setting up DNS records for my projects.

That's what I did:

 * Use the DynDNS service from the provider's internet box to get a FQDN: _mydomain.myprovider.com_
 * Open port 443 on the firewall 
 * Set up port forwarding of port 443 to the server hosting the /e installation (laptop)
 * Make sure to have a fix DHCP lease for the client machine (laptop)

_What I didn't do (yet) but might be necessary: configure the domain in the home network to point to the internal IP of the host (to avoid routing issues)_

Then I configured a CNAME in Cloudflare in order for my registered domain to point to this FQDN:
```
Type CNAME
Name: e.mydomain.com
Content: mydomain.myprovider.com
Proxy-Status: DNS only
```

According to the /e self-hosting README on gitlab, I also needed:
```
Type: CNAME 
Name: mail.e
Content: mydomain.myprovider.com 
TTL: Auto
Proxy-Status: DNS only (reserved IP)
```
_TODO:  The instructions explicitely say this must be an A record and therefore a CNAME won't do. 
However I don't have a fix IP, so I don't see another way_

```
Type: PTR 
Name: 192.168.1.137 
IP: mail.e 
TTL: Auto
Proxy-Status: DNS only (reserved IP)
```
_TODO: Cloudflare is not the right place to put this entry. The home setup won't work._

Since I was using a VM with virtualbox and vagrant, the VM IP was not accessible from the home network.I needed a port forwarding rule from the host to the guest. To do this, I edited the _Vagrantfile_:
```
Vagrant.configure("2") do |config|
  config.vm.network "forwarded_port", guest: 443, host: 443 
end
```

I exited the VM and re-ran _vagrant up_ and _vagrant ssh_ for the change to be effective.


With this experimental setup in place, it should be possible to run the installation script.

## 4. Install /e self-hosting beta

Get installation script:
```
vagrant@ubuntu-bionic:~$ wget https://gitlab.e.foundation/e/infra/bootstrap/raw/master/bootstrap-generic.sh
```

Run installation script as root as described on the gitlab documentation:
```
vagrant@ubuntu-bionic:~$ sudo su
root@ubuntu-bionic:/home/vagrant# bash bootstrap-generic.sh https://gitlab.e.foundation/e/infra/ecloud-selfhosting master
```

I was prompted to provide information:
 
 * As mailserver (management) domain I provided _e.mydomain.com_
 * As additional domain(s) I did not provide any
 * As alternative email I did not provide any
 * I said I did NOT need OnlyOffice

The installation script then provided a list of additional DNS entries to be configured:
```
MX                    |  e.mydomain.com               |  mail.e.mydomain.com  |  10
CNAME                 |  autoconfig.e.mydomain.com    |  mail.e.mydomain.com  |  -
CNAME                 |  autodiscover.e.mydomain.com  |  mail.e.mydomain.com  |  -
CNAME                 |  spam.e.mydomain.com          |  mail.e.mydomain.com  |  -
CNAME                 |  welcome.e.mydomain.com       |  mail.e.mydomain.com  | 
```

I added the entries to my DNS configuration on Cloudflare.

_BLOCKER: Home setup is not possible. The reverse DNS configuration must be provided by the entity owning the IP ranges, which in the case of a home setup is the internet provider._

Alternatives:

 1. Install the software on a vm hosted by a cloud provider (for example _digitalocean_) 
 2. Try to install nextcloud directly, finding out how to use the /e usermanagement with nextcloud
 3. Ask the /e support if there is a way to install their software skipping Postfix

## Links:

[Install /e on LG G3](https://doc.e.foundation/devices/d855/install)

[Latest dev builds](https://images.ecloud.global/dev/d855)

[Download Vagrant](https://www.vagrantup.com/downloads.html)

[Vagrant: Getting Started](https://www.vagrantup.com/intro/getting-started/index.html)

[Self-hosting](https://gitlab.e.foundation/e/infra/ecloud-selfhosting)

[Cloudflare](https://www.cloudflare.com)
