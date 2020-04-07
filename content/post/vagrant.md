+++
math = true
date = "2020-04-05T15:00:00+02:00"
title = "Try out Cozy on Vagrant VM"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

__Goal:__

_Try out Vagrant: set up a Debian Testing VM. Get Cozy up and running._

Cozy is a free personal cloud platform, self-hostable and written in Go. Since long time I would like to adopt a solution to sync my photographs from my mobile to my server, since I don't like pushing personal photographs directly to a public cloud.

Tasks:

1. Install Vagrant and create Debian Buster VM
2. Install Cozy using the Cozy repositories and the instructions on their website 
3. Use a private network

## 1. Vagrant 

First of all, install linux-headers and virtualbox:
```
eramon@caipirinha:~/dev/vagrant$ sudo apt-get install linux-headers-amd64 
eramon@caipirinha:~/dev/vagrant$ sudo apt-get install virtualbox
```
I downloaded Vagrant ("linux" version) and installed it manually. Using the available debian package installs Vagrant with libvirt, and I wanted to use Vagrant with Virtualbox. The .deb file from the Vagrant downloads page gave me an error when trying to install it with dpkg.
```
eramon@caipirinha:~/dev/vagrant$ wget https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_linux_amd64.zip

eramon@caipirinha:~/dev/vagrant$ unzip vagrant_2.2.7_linux_amd64.zip 
Archive:  vagrant_2.2.7_linux_amd64.zip
  inflating: vagrant                 

eramon@caipirinha:~/dev/vagrant$ sudo mv vagrant /usr/local/bin/
```

Pull and run a Debian Buster VM:
```
eramon@caipirinha:~/dev/vagrant$ vagrant init debian/buster64
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.

eramon@caipirinha:~/dev/vagrant$ vagrant up 
...
```
It didn't work:
```
==> default: Rsyncing folder: /home/eramon/dev/vagrant/ => /vagrant
There was an error when attempting to rsync a synced folder.
Please inspect the error message below for more info.

Host path: /home/eramon/dev/vagrant/
Guest path: /vagrant
Command: "rsync" "--verbose" "--archive" "--delete" "-z" "--copy-links" "--no-owner" "--no-group" "--rsync-path" "sudo rsync" "-e" "ssh -p 2222 -o LogLevel=FATAL   -o ControlMaster=auto -o ControlPath=/tmp/vagrant-rsync-20200405-14231-1enocwj -o ControlPersist=10m  -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i '/home/eramon/dev/vagrant/.vagrant/machines/default/virtualbox/private_key'" "--exclude" ".vagrant/" "/home/eramon/dev/vagrant/" "vagrant@127.0.0.1:/vagrant"
Error: Auto configuration failed
140440902825696:error:25066067:DSO support routines:DLFCN_LOAD:could not load the shared library:dso_dlfcn.c:185:filename(libssl_conf.so): libssl_conf.so: cannot open shared object file: No such file or directory
140440902825696:error:25070067:DSO support routines:DSO_load:could not load the shared library:dso_lib.c:244:
140440902825696:error:0E07506E:configuration file routines:MODULE_LOAD_DSO:error loading dso:conf_mod.c:285:module=ssl_conf, path=ssl_conf
140440902825696:error:0E076071:configuration file routines:MODULE_RUN:unknown module name:conf_mod.c:222:module=ssl_conf
rsync: connection unexpectedly closed (0 bytes received so far) [sender]
rsync error: error in rsync protocol data stream (code 12) at io.c(235) [sender=3.1.3]
```

I found following workaround online to avoid the error:
```
eramon@caipirinha:~/dev/vagrant$ export OPENSSL_CONF=/etc/ssl/
```

_TODO: Report Debian bug?_

After that, I was able to connect to the new VM:
```
eramon@caipirinha:~/dev/vagrant$ export OPENSSL_CONF=/etc/ssl/
eramon@caipirinha:~/dev/vagrant$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Checking if box 'debian/buster64' version '10.3.0' is up to date...
==> default: Machine already provisioned. Run `vagrant provision` or use the `--provision`
==> default: flag to force provisioning. Provisioners marked to run always will still run.

==> default: Machine 'default' has a post `vagrant up` message. This is a message
==> default: from the creator of the Vagrantfile, and not from Vagrant itself:
==> default: 
==> default: Vanilla Debian box. See https://app.vagrantup.com/debian for help and bug reports
eramon@caipirinha:~/dev/vagrant$ vagrant ssh
Linux buster 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1 (2020-01-26) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
vagrant@buster:~$ 
```
Inside the vm, it's possible to execute root command with sudo under the _vagrant_ user.

## 2. Cozy
On a first try, I'm going the easy way. I'm using the cozy repositories to install the software exactly as described in the website. Later, I would like to:
a) User standard (and if possible up to date) software packages
b) Containerize the application (as I did with an older version of cozy, a couple of years ago, the first time I tried the application)

The instructions on the website will install cozy, couchdb and nginx.

Install curl:
```
vagrant@buster:~$ sudo apt-get update
vagrant@buster:~$ sudo apt-get install curl
```

Install nodejs 12 and activate NodeSource repository:
```
vagrant@buster:~$ curl -sL https://deb.nodesource.com/setup_12.x | bash -
vagrant@buster:~$ sudo apt-get install -y nodejs
```

Install cozy required packages:
```
vagrant@buster:~$ sudo apt install ca-certificates apt-transport-https wget
```

Fetch GPG Cozy signing key:
```
vagrant@buster:~$ wget https://apt.cozy.io/cozy-keyring.deb
vagrant@buster:~$ sudo dpkg -i cozy-keyring.deb
```

Add cozy repositories:
```
vagrant@buster:~$ sudo su
root@buster:/home/vagrant# echo "deb https://apt.cozy.io/debian/ buster testing" > /etc/apt/sources.list.d/cozy.list
root@buster:/home/vagrant# exit
vagrant@buster:~$ sudo apt update
```

Install CouchDB:
```
vagrant@buster:~$ sudo apt install cozy-couchdb
```
Choose 'standalone'
Set all passwords to 'supersecret'

```
vagrant@buster:~$ curl http://localhost:5984/
{"couchdb":"Welcome","version":"2.3.1","git_sha":"c298091a4","uuid":"3d67b20f20099bf5c92a15e3e8758a51","features":["pluggable-storage-engines","scheduler"],"vendor":{"name":"The Apache Software Foundation"}}
```

Install cozy-stack:
```
vagrant@buster:~$ sudo apt install cozy-stack
```
Cozy user 'cozy'. Admin user 'admin'. Set all paswords to 'supersecret'.

Cozy stack is up and running:
```
vagrant@buster:~$ curl http://localhost:8080/version
{"build_mode":"production","build_time":"2020-03-09T14:00:00Z","runtime_version":"go1.13.5","version":"2:1.4.8"}
```

Enable user namespaces and set permanently:
```
agrant@buster:~$ sudo sysctl -w kernel.unprivileged_userns_clone=1
kernel.unprivileged_userns_clone = 1
vagrant@buster:~$ sudo su
root@buster:/home/vagrant# echo 'kernel.unprivileged_userns_clone=1' > /etc/sysctl.d/99-cozy.conf
root@buster:/home/vagrant# exit
```

Finally, install cozy:
```
vagrant@buster:~$ sudo apt install cozy
```

## 3. Private network
All instructions I found for installing and self-hosting Cozy include the configuration of a public URL and the corresponding DNS records. However, I don't want to expose this service to the Internet, I just want to use Cozy to sync my devices directly to my server. So I tried to figure out if Cozy works as well configuring nginx with a self-signed certificate. 

_TODO_

## Links:
[Debian wiki: Vagrant Quick Start](https://wiki.debian.org/Teams/Cloud/VagrantQuickStart)

[Download Vagrant](https://www.vagrantup.com/downloads.html)

[Vagrant: Getting Started](https://www.vagrantup.com/intro/getting-started/index.html)

[Install Cozy on Debian Server](https://docs.cozy.io/en/tutorials/selfhost-debian/)

[Activate NodeSource repository](https://github.com/nodesource/distributions#user-content-installation-instructions) 

[Cozy - ArchWiki] (https://wiki.archlinux.org/index.php/Cozy)

