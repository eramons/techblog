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

As a server I wasn't so sure. I would have liked to host /e on my K3s cluster on the Raspberry Pi 4, but unfortunately arm was not supported at the time, only x86 or x86-64. So I finished up setting up a hosted Ubuntu VM on the Digital Ocean Developer Cloud. 

_NOTE: First I thought about running the /e software on a vm directly on my laptop, using Vagrant as virtualization environment.  Then I found out it's not possible to run the installation script on the home network, since the DNS configuration requires a reverse DNS record, which must be set up by the owner of the IP ranges, impossible to achieve with the home network._  

 * Client side: install /e operating system on LG G3 d855 (International) 
 * Server side: install /e self-hosting on a Ubuntu VM hosted by Digital Ocean 

For the self-hosting, I also needed:
 
 * A hosted public domain (_mydomain.com_)
 * A DNS provider to register the necessary DNS entries (_Cloudflare_)

Assuming all the pre-conditions were met, following milestones had to be reached:

 1. Install /e on the mobile phone
 2. Create a Digital Ocean account and set up an Ubuntu VM 
 3. Set up DNS
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

## 2. Set up an Ubuntu VM 

Before having a server to install the /e self-hosting software on, I decided to try it out running it on a hosted virtual machine. As cloud service I would use _DigitalOcean_, since I have been wanting to try it out for a while. 

After registering, I chose "Ubuntu Server" as "Starting Point". Servers in Digital Ocean are called _Droplets_, so I was about to create my first droplet:

 * $5.00 Droplet - 1GB / 1 CPU 25 GB SSD Disk 

_Droplets are virtual machines. A virtual machine acts just like a standalone computer, so it runs an operating system (ours are all Linux-based) which you can log into in order to install and run software._

1 GB RAM is not enough according to the requirements on the documentation. I resized the VM to 2 GB RAM. 
After setting up the VM, I accessed via "Console" as root and created a non-root user. SSH is enabled and running. The VM gets a public IPv4 IP address. 

## 3. DNS

For self-hosting /e you need to have at least a domain registered. I already own _mydomain.com_ and I use Cloudflare for setting up DNS records for my projects.

I configured an A record in Cloudflare in order for my registered domain to point to the VM:
```
Type: A 
Name: e.mydomain.com
Content: <Public IP Address>  
Proxy-Status: DNS only
```

According to the /e self-hosting README on gitlab, I also needed:
```
Type: A 
Name: mail.e.mydomain.com
Content: <Public IP Address> 
Proxy-Status: DNS only
```

Still following the instructions, a reverse DNS entry must be set on the _VPS settings on the hoster's website_:
```
Type: PTR
<Public IP Address> -> mail.e.mydomain.com.
```

Logged in to my DigitalOcean account, I found under _Networking_ -> _PTR records_ the following:
```
You have no PTR records.
DigitalOcean will automatically create a PTR record for a server when you rename the host Droplet to the fully qualified domain name of a domain you are managing on your account.
```

So I changed the hostname of the VM to _mail.e.mydomain.com_. I also changed the hostname on the VM itself:
```
root@ubuntu-s-1vcpu-1gb-fra1-01:/home/eramon# echo "mail.e.mydomain.com" > /etc/hostname
```

Then I rebooted. After logging in, it looked like the change was effective:
```
eramon@mail:~$
```

Now I saw following PTR record under "Networking" -> "PTR records":
```
IP Address: <Public IP Address> 
PTR Record: mail.e.mydomain.com. 
```

With this setup in place, it should be possible to run the installation script.

## 4. Install /e self-hosting beta

Get installation script:
```
eramon@mail:~$ wget https://gitlab.e.foundation/e/infra/bootstrap/raw/master/bootstrap-generic.sh
```

Run installation script as root as described on the gitlab documentation:
```
eramon@mail:~$ sudo bash bootstrap-generic.sh https://gitlab.e.foundation/e/infra/ecloud-selfhosting master
```

I was prompted to provide information:
 
 * As mailserver (management) domain I provided _e.mydomain.com_
 * As additional domain(s) I did not provide any
 * I provided an alternative e-mail address _eramon@mydomain.com_ 
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

_TODO: Current issue:_
```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

I asked the /e community support for assistance. According to an older thread, the issue is related to the fact that in later builds MariaDB does not allow root access without password. 

## Other approaches and conclussion:

On the server side, as for the last update of this post, there is still an open issue which prevents the install script to finish succesfully.

Unfortunately, the instructions and code for the self-hosting installation do not allow for flexibility. The installation script downloads source and files and applies a default configuration using _salt_. Then it uses docker-compose to run docker instances of the different applications.

On the client side, the android-based OS looks really good. The synchronization of files and photographs works perfectly and it's uncomplicated. I liked it.

On the server side, for me this setup is fine as experimental, but a no go for a potential productive setup. Reasons:

 * I don't want to host any data on the cloud. The idea is to keep private files on the home network
 * I don't want to host an own mail server. I'm not interested in having an own e-mail address for this  * I'm just interested on files and pictures synchronization.

## Next steps:

Basing on the premise that I don't need either a mail server nor only office, a setup with just NextCloud on the server side would be enough for me. I got the recommendation from the /e support to do so. Apparently there is no special account management to take into consideration, just installing nextcloud and connecting the LG G3 to the server should do. 

New post - Nextcloud - coming soon :)


## Links:

[Install /e on LG G3](https://doc.e.foundation/devices/d855/install)

[Latest dev builds](https://images.ecloud.global/dev/d855)

[Digital Ocean](https://www.digitalocean.com/)

[Self-hosting](https://gitlab.e.foundation/e/infra/ecloud-selfhosting)

[Cloudflare](https://www.cloudflare.com)

## Troubleshooting and /e community support

[PHP Fatal Error: apc_mmap](https://community.e.foundation/t/php-fatal-error-apc-mmap/24440?u=eramon)

[MariaDB issue](https://community.e.foundation/t/mariadb-issue-access-denied-for-user-root-localhost/24500)
