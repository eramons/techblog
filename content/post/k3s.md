+++
math = true
date = "2020-06-15T15:00:00+02:00"
title = "K3s on Raspberry Pi 4"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

__Goal:__

_Set up a minimal Kubernetes cluster on Rapberry Pi 4._

I ordered a new Raspberry Pi 4 a couple of days ago. I already use one at work for automated testing and I think it's pretty cool, but I actually wasn't sure what I wanted it for. After giving it a thought, I decided to install Rancher's K3S distribution on it, turning it to a convenient, low-power-consumption, single-node K8s distribution I can use as a playground.


To try it out, I would deploy the manifests of my kubecozy project on it.

_Possibility 1: the Raspberry Pi 4 will be both master and worker_
_Possibility 2: the Raspberry Pi 4 will be the master. An older Raspberry Pi 3 will be the worker_

Willing to simplify, I went for the first possibility. 

![Raspberry Pi 4 with case](/techblog/img/raspberrypi.jpg)

In addition to the Raspberry 4, I will use: 

*  Power adapter
*  Ethernet cable
*  SD card

__Tasks:__

1. Install latest Raspbian on the Raspberry Pi 4 
2. Install K3S 
3. Enable 64 bits kernel 
4. Fix up networking 
4. Set up the cluster
5. Deploy something 

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
```
_NOTE: default username/password is pi/raspberry_

## 2. Install K3S 

Get K3S latest binary (make sure to get the armhf version):
```
pi@raspberrypi:~ $ wget https://github.com/rancher/k3s/releases/download/v1.17.4%2Bk3s1/k3s-armhf
```

Rename binary and change file mode to make it executable:
```
pi@raspberrypi:~ $ sudo mv k3s-armhf /usr/local/bin/k3s
pi@raspberrypi:~ $ chmod +x /usr/local/bin/k3s
```

Avoid having to use sudo to invoke k3s:
```
pi@raspberrypi:~ $ sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

The container runtime i.e. docker is automatically installed with K3s. Don't install it manually or this will mess up the iptables configuration. 

_Note: I actually did that mistake. The sympthom was that the pods weren't able to communicate with each other. I was able to fix it by uninstalling iptables, uninstalling docker.io, installing nftables and restarting._ 

Show cluster info:
```
pi@raspberrypi:~ $ k3s kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

Run k3s as a daemon:
```
pi@raspberrypi:~ $ sudo vi /etc/systemd/system/k3s.service
```

The content of the file should look like this:
```
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
After=network-online.target

[Service]
Type=notify
ExecStart=/usr/local/bin/k3s server
KillMode=process
Delegate=yes
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Restart the service:
```
pi@raspberrypi:~ $ sudo systemctl daemon-reload
pi@raspberrypi:~ $ sudo systemctl start k3s
```

In order for the service to start automatically after a reboot, you have to enable the service as follows:
```
pi@raspberrypi:~ $ sudo systemctl enable k3s
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /lib/systemd/system/k3s.service.
```

## 3. Networking

Enable wireless networking:
```
pi@raspberrypi:~ $ sudo raspi-config 
```
Follow the instructions to activate the wifi network interface and get the new IP. Afterwards, configure a  static lease in the home network settings so the raspi will always get the same address. 

## 4. Enable 64 bits kernel

I found out many available docker images are available for arm 64 bits only. The raspi 4 supports 64 bits, however for Raspbian there was still only the 32-bits version available. After a little research I found a guide to enable a 64 bits kernel on the 32-bits distribution. 

Enable 64 bits:
```
pi@raspberrypi:~ $ sudo su
root@raspberrypi:/home/pi# echo 'arm_64bit=1' >> /boot/config.txt
```

After the modification:
```
pi@raspberrypi:~ $ uname -a
Linux raspberrypi 4.19.97-v8+ #1294 SMP PREEMPT Thu Jan 30 13:27:08 GMT 2020 aarch64 GNU/Linux
```

Reboot the raspberry pi and disconnect the ethernet cable. Afterwards, the k3s service will start automatically and the device will be accessible over wi-fi.

## 5. Set up the cluster

I oriented myself on previous work building a home-made cluster to know which do I have to set up: 

 * Install cert-manager
 * Set-up nfs for persistent volumes _The package nfs-common was already available on raspbian._
 * Set-up ClusterIssuer
 * Network overlay _Flannel was already automatically installed when running the K3s installation script._

### 5.1. Install cert-manager

Install cert-manager using manifests:
```
pi@raspberrypi:~/kubecozy $ sudo k3s kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.yaml
```

Check that it's running:
```
pi@raspberrypi:~/kubecozy $ sudo k3s kubectl get pods -n cert-manager
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-77fcb5c94-rz8kd              1/1     Running   0          64s
cert-manager-cainjector-8fff85c75-nsgsx   1/1     Running   0          64s
cert-manager-webhook-54647cbd5-dtlnk      1/1     Running   0          63s
```
It seemed to be working.

### 5.2 Set-up kubectl client

To avoid having to ssh to the device to manage the cluster, I copied kubeconfig content from _/etc/rancher/k3s/k3s.yaml_ and insert it into _/home/eramon/.kube/config_ to be able to access the cluster directly from my client machine. Remember to change the IP as necessary inside the file.

_NOTE: I already had kubectl installed_

## 6. Deploy example application

To try out the cluster, I wanted to deploy my kubecozy project (see [Home-made K8s cluster]({{< ref "/post/kubernetes_cluster" >}}))

 If I did it right the first time, to deploy the same application in another cluster should be quite straightforward.


So I checked out my previous work:
```
eramon@caipirinha:~/dev$ git clone https://github.com/eramons/kubecozy.git
eramon@caipirinha:~/dev$ cd kubecozy
```

However there were still some modifications and changes I had to go through due to the new architecture and the new distribution:

 * Modify ingress to use traefik instead of nginx
 * Build a new cozy-stack docker image for arm and find arm versions for mailhog and couchdb

And some customizations which are deployment-dependent and therefore not part of the manifests:

 * Set up the cluster issuer
 * Set up the cloudflare token
 * Set up the persistent volumes and persistent volume claims
 * Generate the secrets

### 6.1. Remove nginx and set up ingress

In K3s, Traefik comes already installed as default ingress controller:
```
eramon@caipirinha:~/dev/kubecozy$ kubectl get svc traefik --namespace kube-system -w
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
traefik   LoadBalancer   10.43.213.9   192.168.1.131   80:31368/TCP,443:30018/TCP   213d
```

Change kustomization file to remove the nginx ingress: [arm64v8/kustomization.yaml](https://github.com/eramons/kubecozy/blob/master/arm64v8/kustomization.yaml)

Change the cozy-ingress.yaml.example removing the nginx annotations: [arm64v8/cozy-ingress.yaml.example](https://github.com/eramons/kubecozy/blob/master/arm64v8/cozy-ingress.yaml.example)

Customize cozy-ingress.yaml, changing the hostnames:
```
eramon@caipirinha:~/dev/kubecozy$ cp cozy-ingress.yaml.example cozy-ingress.yaml
eramon@caipirinha:~/dev/kubecozy$ vi cozy-ingress.yaml
```

### 6.2 New docker images for arm

The docker images I used originally do not work on an arm architecture. I had to do some more work here:

* Create a new cozy-stack docker image for arm64v8. Build it on the raspi and push it to dockerhub.
* Change the deployment file to use the new image instead of the amd64 one: _eramon/cozy-stack-arm64v8_
* Change the couchdb deployment file to use the arm64 version: _arm64v8/couchdb:3.1.0_
* Change the mailhog deployment file to use an arm64 version: _albertsai/mailhog_

New deployment manifests:
[arm64v8](https://github.com/eramons/kubecozy/blob/master/arm64v8)

New Dockerfile for cozy-stack on arm64v8:
[arm64v8/docker/Dockerfile](https://github.com/eramons/kubecozy/blob/master/arm64v8/docker/Dockerfile)

### 6.3 Cloudflare token

Customize cloudflare.yaml, providing the cloudflare api token:
```
eramon@caipirinha:~/dev/kubecozy$ cp cloudflare.yaml.example cloudflare.yaml
eramon@caipirinha:~/dev/kubecozy$ vi cloudflare.yaml
```

The cloudflare.yaml is not part of the kustomize file, since is cluster specific. So I had to apply:
```
eramon@caipirinha:~/dev/kubecozy$ kubectl apply -f cloudflare.yaml
```

### 6.4 Cluster issuer

Customize letsencrypt-prod.yaml, changing the e-mail address:
```
eramon@caipirinha:~/dev/kubecozy$ cp letsencrypt-prod.yaml.example letsencrypt-prod.yaml
eramon@caipirinha:~/dev/kubecozy$ vi letsencrypt-prod.yaml
```

### 6.5 Persistent volumes 

Customize pv files, modifying IP, path and size:
```
eramon@caipirinha:~/dev/kubecozy$ cp couchdb-nfs-pv.yaml.example couchdb-nfs-pv.yaml
eramon@caipirinha:~/dev/kubecozy$ vi couchdb-nfs-pv.yaml
```

```
eramon@caipirinha:~/dev/kubecozy$ cp cozy-nfs-pv.yaml.example cozy-nfs-pv.yaml
eramon@caipirinha:~/dev/kubecozy$ vi cozy-nfs-pv.yaml
```

The persistent volumes are not part of kustomize (since there are cluster specific), so I had to apply the manifests:
```
eramon@caipirinha:~/dev/kubecozy$ kubectl apply -f cozy-nfs-pv.yaml
eramon@caipirinha:~/dev/kubecozy$ kubectl apply -f couchdb-nfs-pv.yaml
```

### 6.6 Secrets

The secrets must be generated manually, following the same process described on my previous post, using the new cozy-docker arm64v8 image instead of the amd64 one:

```
eramon@caipirinha:~/dev/kubecozy$ touch cozy-admin-passphrase
eramon@caipirinha:~/dev/kubecozy$ sudo docker run --rm -it -p 8080:8080 -v /home/pi/kubecozy/cozy-admin-passphrase:/home/cozy/cozy-admin-passphrase eramon/cozy-stack-arm64v8 cozy-stack config passwd cozy-admin-passphrase
eramon@caipirinha:~/dev/kubecozy$ kubectl create secret generic cozy-admin-passphrase-secret --from-file=cozy-admin-passphrase
```

```
eramon@caipirinha:~/dev/kubecozy$ mkdir vault
eramon@caipirinha:~/dev/kubecozy$ sudo docker run --rm -it -p 8080:8080 -v /home/pi/kubecozy/vault:/home/cozy/vault eramon/cozy-stack-arm64v8 cozy-stack config gen-keys /home/cozy/vault
eramon@caipirinha:~/dev/kubecozy$ kubectl create secret generic cozy-vault-secret --from-file=vault/vault.enc --from-file=vault/vault.dec
```

```
eramon@caipirinha:~/dev/kubecozy$ echo -n "admin" > dbusername
eramon@caipirinha:~/dev/kubecozy$ echo -n `pwgen -s -1 16` > dbpassphrase
eramon@caipirinha:~/dev/kubecozy$ kubectl create secret generic db-secret --from-file=dbusername --from-file=dbpassphrase
```

And finally:
```
eramon@caipirinha:~/dev/kubecozy$ k3s kubectl apply -k .
```

The cozy application previously deployed on my home-made bare-metal K8s cluster was now working on the K3s cluster on the Raspberry Pi 4, using the same manifests - after the minimal modifications described.

## Links

[Lightweit Kubernetes](https://k3s.io/)

[K3S Releases](https://github.com/rancher/k3s/releases/tag/v1.17.4+k3s1)

[Issue with iptables](https://github.com/rancher/k3s/issues/1597)

[Kubernetes on Raspberry Pi with K3s](https://www.michaelburch.net/blog/Kubernetes-on-Raspberry-Pi-with-K3s.html)
