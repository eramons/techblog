+++
math = true
date = "2020-03-29T15:00:00+02:00"
title = "K3S Cluster on Raspberry Pi 4"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

__Goal:__

_Set up a minimal Kubernetes cluster on Rapberry Pi 4._

I ordered a new Raspberry Pi 4 a couple of days ago. I already use one at work for automated testing and I think it's pretty cool, but I actually wasn't sure what I wanted it for. After giving it a thought, I decided to try to set up a K3S cluster on it.

_Possibility 1: the Raspberry Pi 4 will be both master and worker_
_Possibility 2: the Raspberry Pi 4 will be the master. An older Raspberry Pi 3 will be the worker_

Let's start with the first possibility and see if later it's possible to expand. 

In addition to the Raspberry 4, I will use: 

*  Power adapter
*  Ethernet cable
*  SD card

__Tasks:__

1. Install latest Raspbian on the Raspberry Pi 4 
2. Install K3S 

## 1. Install Raspbian 
Download latest Raspbian image. 

Unzip:
```
eramon@caipirinha:~/dev/raspberrypi$ unzip 2020-02-13-raspbian-buster-lite.zip 
Archive:  2020-02-13-raspbian-buster-lite.zip
  inflating: 2020-02-13-raspbian-buster-lite.img  
```

Copy to the sdcard:
```
eramon@caipirinha:~/dev/raspberrypi$ sudo dd if=2020-02-13-raspbian-buster-lite.img of=/dev/sdb bs=4MB
462+1 records in
462+1 records out
1849688064 bytes (1.8 GB, 1.7 GiB) copied, 133.426 s, 13.9 MB/s

eramon@caipirinha:~/dev/raspberrypi$ sudo sync
```
_NOTE: find out the device name first, using dmesg. For me, it was /dev/sdb._


To make sure we can connect to the Raspberry Pi 4 via ssh, we need to manually add a _SSH_ file to the SDCard. 

Mount the sdcard first partition and add the file:
```
eramon@caipirinha:~/dev/raspberrypi$ sudo mount /dev/sdb1 mnt/
eramon@caipirinha:~/dev/raspberrypi$ cd mnt/
eramon@caipirinha:~/dev/raspberrypi/mnt$ sudo touch SSH
eramon@caipirinha:~/dev/raspberrypi/mnt$ cd ..
eramon@caipirinha:~/dev/raspberrypi$ sudo umount mnt
eramon@caipirinha:~/dev/raspberrypi$ sudo sync
``` 

Insert the SD Card in the Raspberry Pi 4. Connect the power adapter and the ethernet cable.  

Next, I have to find out the IP of the Raspberry Pi 4. The easiest way was to connect to my home router and check the DHCP leases list.

Once found out, connect to the Raspberry Pi 4 over ssh:
```
eramon@caipirinha:~/dev/raspberrypi$ ssh pi@192.168.1.134
The authenticity of host '192.168.1.134 (192.168.1.134)' can't be established.
ECDSA key fingerprint is SHA256:VyBXTNZqpeU9LfG0bCiOjfplOON7nkLs4+k0z9toz7s.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.134' (ECDSA) to the list of known hosts.
pi@192.168.1.134's password: 
Linux raspberrypi 4.19.97-v7l+ #1294 SMP Thu Jan 30 13:21:14 GMT 2020 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.


Wi-Fi is currently blocked by rfkill.
Use raspi-config to set the country before use.

pi@raspberrypi:~ $ 
```
_NOTE: default username/password is pi/raspberry_

## 2. Install K3S 

Get K3S latest binary (make sure to get the armhf version):
```
pi@raspberrypi:~ $ wget https://github.com/rancher/k3s/releases/download/v1.17.4%2Bk3s1/k3s-armhf
...
k3s-armhf                 100%[====================================>]  45.88M  4.48MB/s    in 16s     

2020-03-29 13:50:16 (2.93 MB/s) - ‘k3s-armhf’ saved [48103424/48103424]
```

Rename binary and change file mode to make it executable:
```
pi@raspberrypi:~ $ mv k3s-armhf k3s
pi@raspberrypi:~ $ chmod +x k3s-armhf 
```

Run the server:
```
pi@raspberrypi:~ $ sudo ./k3s server &
```

Show cluster info:
```
pi@raspberrypi:~ $ sudo ./k3s kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

_LATER:_
To add new nodes to the cluster, first find out the node token:
```
pi@raspberrypi:~ $ sudo cat /var/lib/rancher/k3s/server/node-token
K10508ae11bab8e277893f008cc9cd4107b2acff73682d8e71d7eb8d7221e2c97a6::server:cd7d5d5dd18b4bc4fa7390194078c9d0
```

The new node should use the following command to join the cluster (not tested yet):
```
pi@raspberrypi:~ $ sudo ./k3s agent --server https://localhost:6443 --token K10508ae11bab8e277893f008cc9cd4107b2acff73682d8e71d7eb8d7221e2c97a6::server:cd7d5d5dd18b4bc4fa7390194078c9d0
```




## Links
[Lightweit Kubernetes](https://k3s.io/)

[K3S Releases](https://github.com/rancher/k3s/releases/tag/v1.17.4+k3s1)
