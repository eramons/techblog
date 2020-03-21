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

Home Kubernetes Cluster with old hardware: 1 Master and 1 Worker 

__Goal:__

_Set up a Home Kubernetes cluster_

Tasks:

1. Find old hardware
2. Install Ubuntu Server 18.04 LTS 
3. Set up Master Node
4. Set up Worker Node 

## 1. Find old hardware 

Since kubernetes is lightweight and can run almost everywhere, I decided to go down to the cellar and rescue some old PC which I thought could still work for this. That's what I found:

* A MAC Mini Server from year 2009. Two 500GB disks. A still working grub bootloader told me I had a MacOS, an Ubuntu and two Debian distributions installed on the machine. This computer had 4GB RAM. I removed one of the disks, which was broken. One 500GB disk is more than enough for the Master anyway.
* A PC tower whose components were bought separately and which I proudly assembled myself in 2007. Refurbished through the years, this PC had 6GB RAM and three hard disks: 250GB, 500GB and 2TB. Over the two first disks I had a LVM system installed, with a Debian installation and two separated home partitions. The PC didn't boot anymore, since the first disk - hosting the Master Boot Record - was broken. I removed the broken disk and the second LVM disk, keeping only the 2TB disk on the machine. 

So I thought the MAC Mini should be my Master and the PC Tower should be my Worker :) 

## 2. Install Ubuntu Server LTS

I decided to install the newest Ubuntu Server version with Long Term Support (LTS) on both machines. An alternative could have been to install the latest Debian stable.

Download Ubuntu Server 18.04.4 LTS from https://ubuntu.com/download/server

For the MacMini, I just created a bootable USB.

Insert an USB drive and find out which device is it mapped to:
```
sudo dmesg |grep sd
...
[24400.755280] sd 4:0:0:0: [sdb] Attached SCSI disk
[24406.001628] EXT4-fs (sdb1): mounted filesystem with ordered data mode. Opts: (null)
[39200.249434] sd 4:0:0:0: [sdb] 31266816 512-byte logical blocks: (16.0 GB/14.9 GiB)
...
```
It was /dev/sdb. So I copied the downloaded image to the USB drive:
```
sudo dd if=ubuntu-18.04.4-live-server-amd64.iso of=/dev/sdb
```

For the old PC, I had to burn a DVD, since it was so old it did not even boot from USB. For that I borrowed a quite old Windows laptop which still had a DVD-drive. I burned the ISO image using the Windows Disc Image Burner.

After having the media prepared, I installed Ubuntu Server on both machines. In both cases, I chose "Manual" for setting up the disks and the partitions, using the whole disk. 

NOTE: swap is not supported. Either it must be omitted during installation, or it must be switched off afterwards:
```
sudo swapoff -a 
```
## 3. Master node

### 3.1. Install runtime
I searched for a suitable container runtime to install, gettin two results which made sense for me:
```
eramon@caipirinha:~/dev/techblog$ apt-cache search docker.io
containerd - open and reliable container runtime
docker.io - Linux container runtime
...
```

Although containerd would be enough, I installed docker.io since having the extra docker tools for debugging or whatever might be useful later:
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

### 3.3 Initialise master node
The goal was to set up a single control-plane cluster with kubeadm.
```
kubeadmin init --pod-network-cidr=10.244.0.0/16
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.129:6443 --token jn333p.q9hskm01cs12iak7 \
    --discovery-token-ca-cert-hash sha256:0966963ed31ac9d898e3d49d154e2f6ed78931f356af5d6c35616ee75585c2f9 
```

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
eramon@pacharan:~$ sudo chown eramon:eramon .kube/config
```

After these steps, we see that kubectl cluster-info is showing something so kubectl is interacting with our new cluster:
```
eramon@pacharan:~$ kubectl cluster-info
Kubernetes master is running at https://192.168.1.129:6443
KubeDNS is running at https://192.168.1.129:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### 3.4 Install a Pod network add-on
As the output of kubeadm said, we must deploy a pod network to the cluster, so the pods can communicate with each other.
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

Show all pods from all namespaces:
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
Flannel was there.


## 4. Worker node

### 4.1 Installation
I followed exactly the same process described in 3.1 and 3.2 to install the container runtime and the kubernetes software on the worker node.

### 4.2 Join the cluster
First enable docker.service (otherwise kubeadm join shows a warning):
```
eramon@whisky:~$ sudo systemctl enable docker.service
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service.
```

To join the cluster, we execute the command extracted from the output of kubeadm before:
```
eramon@whisky:~$ sudo kubeadm join 192.168.1.129:6443 --token jn333p.q9hskm01cs12iak7 --discovery-token-ca-cert-hash sha256:0966963ed31ac9d898e3d49d154e2f6ed78931f356af5d6c35616ee75585c2f9 
W0321 12:59:20.688335   11019 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
```

That's all. With this, the cluster should be ready for the first deployment.

## 5. Open points:

### 5.1. Warning
In both master and worker, I get this warning:
```
[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
```
Not sure what this does exactly mean or how to fix it yet.

## 5.2 Infrastructure as code
I would like to have the whole process scripted and/or described in yaml files and these pushed to github, so the environment can be set up at any time.

## References and useful or interesting links:

[Install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm)

[Create single-control node cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm)

[Install pod network](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

[Flannel](https://github.com/coreos/flannel)
