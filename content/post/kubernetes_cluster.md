+++
math = true
date = "2020-03-20T10:00:00+02:00"
title = "Setup a Home Kubernetes Cluster"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

Kubernetes Bla

__Goal:__

_Set up a Home Kubernetes cluster_

Tasks:

1. Find old hardware
2. Install Ubuntu Server 18.04 LTS 
3. Set up Master Node
4. Set up Worker Node 

## 1. Find old hardware 

Since kubernetes is lightweight and can run almost everywhere, I decided to go down to the cellar and rescue some old PC which I thought could still work for this. That's what I found:

* A MAC Mini Server from year 2009. Two 500GB disks. A still working grub bootloader told me I had a MacOS, an Ubuntu and two Debian distributions set up on the machine. This computer has 4GB RAM. I removed one of the disks, which was broken. 
* A PC tower whose components were bought separately and which I proudly assembled myself in 2007. Refurbished through the years, this PC had 6GB RAM and three hard disks: 250GB, 500GB and 2TB. Over the two first disks I had LVM set up. The PC didn't boot anymore, since the first disk - hosting the Master Boot Record - was broken. I removed the broken disk and the other disk part of the LVM and let just the 2TB one on the machine. 

The MAC Mini would be my Master and the PC Tower would be my Worker :) 

## 2. Install Ubuntu 10.04 LTS

Not much to report here. I decided to install the newest Ubuntu Server version with Long Term Support (LTS) on both machines. 

For the MacMini, I just created a bootable USB:
```
```
Download Ubuntu Server 18.04.4 LTS from https://ubuntu.com/download/server
Insert an USB drive and find out which device is it mapped to:
```
sudo dmesg |grep sd
...
[24400.755280] sd 4:0:0:0: [sdb] Attached SCSI disk
[24406.001628] EXT4-fs (sdb1): mounted filesystem with ordered data mode. Opts: (null)
[39200.249434] sd 4:0:0:0: [sdb] 31266816 512-byte logical blocks: (16.0 GB/14.9 GiB)
...
```
For me, it was /dev/sdb. Now copy the downloaded image to the USB drive:
```
sudo dd if=ubuntu-18.04.4-live-server-amd64.iso of=/dev/sdb
```

For the old PC, I had to burn a DVD, since it did not boot from USB. For that I borrowed a not-so-new Windows laptop which still had a DVD-drive. I burned the ISO image using the Windows Disc Image Burner.

```
dd if=ubuntu-18.0.4-live-server-amd64.iso of=/dev/sdb
```

After having the media prepared, I installed Ubuntu Server on both machines. In both cases, I chose "Manual" for setting up the disks and the partitions, using the whole disk. 

## 3. Install kubernetes on master node

### 3.1. Install runtime
When doing apt-cache search docker.io I see two different options which make sense for me:
docker-containerd
docker.io

Although containerd would be enough, I install docker.io in case the extra tools are handy to do debugigging or whatever later:
```
sudo apt-get install docker.io
```
### 3.2. Install kubernetes software
Install kubeadm, kubectl and kubelet
```
sudo apt-get update
sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo vi /etc/apt/sources.list.d/kubernetes.list
Add:
deb https://apt.kubernetes.io/ kubernetes-xenial main (probably not good, I have bionic) There is no kubernetes-bionic under packages.cloud.google.com/apt/dists
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

kubelet, kubeadm and kubectl are set on hold (to avoid automatic updates, I think)

After installation:
```
systemctl daemon-reload
systemctl restart kubelet
```

## 5. Set up a single control-plane cluster with kubeadm

### 5.1 Add subtitle here

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
eramon@pacharan:~$ sudo chown eramon:eramon .kube/config

After these steps, we see that kubectl cluster-info is showing something so kubectl is interacting with our new cluster:
eramon@pacharan:~$ kubectl cluster-info
Kubernetes master is running at https://192.168.1.129:6443
KubeDNS is running at https://192.168.1.129:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

### 5.2 Install a Pod network add-on
Next step is to install a network overlay, so the pods can communicate with each other.
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network

Install flannel:
https://github.com/coreos/flannel

```
eramon@pacharan:~$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```

```
eramon@pacharan:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-6955765f44-597bj           1/1     Running   0          160m
kube-system   coredns-6955765f44-ctwvs           1/1     Running   0          160m
kube-system   etcd-pacharan                      1/1     Running   0          160m
kube-system   kube-apiserver-pacharan            1/1     Running   0          160m
kube-system   kube-controller-manager-pacharan   1/1     Running   0          160m
kube-system   kube-flannel-ds-amd64-dhc7f        1/1     Running   0          62s
kube-system   kube-proxy-qcv98                   1/1     Running   0          160m
kube-system   kube-scheduler-pacharan            1/1     Running   0          160m
```

### 5.3 Initialise master node
Add text

```
kubeadmin init --pod-network-cidr=10.244.0.0/16

```

## 4. Install kubernetes on worker node
I followed exactly the same process to install kubernetes on the worker node.

References and useful or interesting links:

[Install Kubernetes](https://puri.sm)

[Ubuntu Downloads](https://...)
