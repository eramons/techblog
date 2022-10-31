+++
math = true
date = "2022-10-26T10:00:00+02:00"
title = "Unifi Controller on K3s"
tags = ["home automation","kubernetes"]
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Run the unifi controller as container on my home K3s cluster._

## Introduction

The synology D916+ can run a docker application packed in a single container, which I found quite convenient to host a couple of applications in my home network:

 * The unifi controller, to manage our two Unifi Access Points
 * The Logitech Media Server - yes, I still own several squeezeboxes I wouldn't change for anything :D
 * Home Assistant

Planning to add further applications - and having in mind a setup where the logs from some of the home applications are fed into a elasticsearch cluster later - I decided to move them to Kubernetes, which it meant to reactivate my K3s cluster, which I had installed on a Raspberry Pi 4 a while ago - see my previous post:

[K3s on Raspberry Pi 4]({{< ref "/post/k3s" >}})

Another reason was that I had in mind to try out to centralize the logs and create visualizations from the access points, and later from the home automation software. 
 
__Steps:__ 

   * Update Raspbian
   * Install Helm
   * DNS Resolution
   * Certmanager
   * Clusterissuer
   * Replace the Ingress controller
   * Unifi K8s resources
   * Confiure Unifi Logging

### 1. K3s 

First thing was to power on my rasperry pi 4 using the same SSD I had used the last time I played with k3s, which hosted K3s on a Raspbian operating system. 

![unifi](/techblog/img/unifi.jpg)

 * I used one of the access points to power the raspi via USB :)
 * A micro-HDMI to HDMI adapter is a good idea to do some troubleshooting if something goes wrong and the raspi is not reachable via SSH for some reason

#### 1.1. Raspbian

I needed to upgrade Raspbian from _buster_ to _bullseye_. For this, first I had to upgrade the current system:
```
pi@raspberrypi:~ $ sudo apt-get update
pi@raspberrypi:~ $ sudo apt-get dist-upgrade
```
Once finished, edite _/etc/apt/sources.list_ and replace _buster_ with _bullseye_. Then run _sudo apt-get update_ and _sudo apt-get dist-upgrade_ again.

Restart the raspi:
```
pi@raspberrypi:~$ sudo reboot
``` 
After updating the operating system, I opted for uninstalling k3s and reinstallng it:
```
pi@raspberrypi:~$ sudo /usr/local/bin/k3s-uninstall.sh
pi@raspberrypi:~$ curl -sfL https://get.k3s.io | sh 
```
Avoid sudo when invoking _kubectl_ locally:
```
pi@raspberrypi:~$ sudo mkdir -p /home/pi/.kube
pi@raspberrypi:~$ sudo cp /etc/rancher/k3s/k3s.yaml /home/pi/.kube/config
pi@raspberrypi:~/.kube $ sudo chown $USER /home/pi/.kube/config
pi@raspberrypi:~/.kube $ sudo chmod 600 /home/pi/.kube/config
pi@raspberrypi:~$ export KUBECONFIG=~/.kube/config
```
_NOTE: copy the _.kube/config_ file to another machine and replace localhost with the actual IP to manage the cluster remotely_

#### 1.2. Helm

_Helm is a package manager for Kubernetes. Helm Charts help define, install and upgrade complex Kubernetes applications._

Install helm
```
pi@raspberrypi:~$ mkdir -p unifi/helm
pi@raspberrypi:~$ cd unifi/helm
pi@raspberrypi:~/unifi/helm $ wget https://get.helm.sh/helm-v3.10.1-linux-arm.tar.gz
pi@raspberrypi:~/unifi/helm $ tar -zxvf helm-v3.10.1-linux-arm.tar.gz
pi@raspberrypi:~/unifi/helm $ sudo mv linux-arm/helm /usr/local/bin/helm
pi@raspberrypi:~/unifi/helm $ rm -rf linux-arm
pi@raspberrypi:~/unifi/helm $ cd ..
```

#### 1.3 DNS

In order to have trustworthy SSL certificates issued by Let's Encrypt, you need to meet following conditions:

 * To own a registered valid domain
 * To have a public DNS record pointing _yourhostname.yourdomain_ to the IP of the ingress controller 

_Let's Encrypt is a nonprofit Certificate Authority providing TLS certificates to websites._

#### 1.4 Certmanager

_Certmanager adds certificate and certificate issuers as resource types in K8s clusters, and simplifies the process of obtaining, renewing and using those certificates._

Install the cert-manager manifests:
```
pi@raspberrypi:~/unifi $ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.crds.yaml
```
Install cert-manager with the helm chart:
```
pi@raspberrypi:~/unifi $ helm repo add jetstack https://charts.jetstack.io
pi@raspberrypi:~/unifi $ helm repo update
pi@raspberrypi:~/unifi $ helm install \
   cert-manager jetstack/cert-manager \
   --namespace cert-manager \
   --create-namespace \
   --version v1.10.0 \
   --set rbac.create=true
```
_NOTE: I actually would rather include the installation of cert-manager - and later the ingress controller- in kustomize. But helm is very convenient, specially for try and error: install, uninstall, upgrade and configuration parameters._

#### 1.5 Cluster Issuer

_Issuers, and ClusterIssuers, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests. All cert-manager certificates require a referenced issuer that is in a ready condition to attempt to honor the request._

There are two mechanisms the clusterissuer provides to prove to Let's Encrypt that you indeed own the domain you're claiming is yours:
* _HTTP01 challenges are completed by presenting a computed key, that should be present at a HTTP URL endpoint and is routable over the internet. This URL will use the domain name requested for the certificate_
* _DNS01 challenges are completed by providing a computed key that is present at a DNS TXT record. Once this TXT record has been propagated across the internet, the ACME server can successfully retrieve this key via a DNS lookup and can validate that the client owns the domain for the requested certificate_

I used the DNS Solver, hosting the DNS record in Cloudflare.

I got the api token in the Cloudflare admin console. Then I defined a secret for it:
```
pi@raspberrypi:~/unifi $ cat cloudflare-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: my_api_token
```
The manifest sample is available in my repo:
[cloudflare-secret.yaml.sample](https://github.com/eramons/k3s-home/blob/main/cluster/clusterissuer.yaml.sample)
To use this manifest, replace _my-api-token_ with an actual token and rename the file to _cloudflare-secret.yaml_.

The generation of the secret, I did manually - unlike the other K8s resources later, where I would later use _kustomize_:
```
pi@raspberrypi:~/unifi $ kubectl apply -f cloudflare-secret.yaml
secret/cloudflare-api-token-secret configured
```
The manifest for the cluster issuer is available in my repo:
[clusterissuer.yaml](https://github.com/eramons/k3s-home/blob/main/cluster/clusterissuer.yaml.sample)
To use this manifest, set the _email_ correctly and rename the file to _clusterissuer.yaml_.

Apply the manifest for the clusterissuer:
```
pi@raspberry:~/dev/unifi$ kubectl apply -f clusterissuer.yaml
```
Alternatively, there is a _kustomization.yaml_ available in my repo:
[cluster/kustomization.yaml](https://github.com/eramons/k3s-home/blob/main/cluster/kustomization.yaml)
For deploying the clusterissuer with kustomize, execute _kubectl apply -k ._ instead.

_NOTE: For the cluster issuer, I configured first the staging let's encrypt url. When I changed to the productive url, the certificate was still the one from staging.  I managed to get the productive one after deleting the secret, the certificate and the ingress and re-creating the ingress again._

#### 1.6 Ingress controller 

Traefik comes pre-installed with K3s.

So you can check that the traefik ingress-controller is up and running:
```
pi@raspberrypi:~/unifi $  kubectl get svc traefik --namespace kube-system
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                      AGE
traefik   LoadBalancer   10.43.252.0   192.168.1.71   80:31740/TCP,443:31443/TCP   14m
```
However I had some specific requirements I was not sure how to solve using traefik:
 * Bare metal cluster - actually only one node acting both as master and worker
 * No load balancer
 * Instal ingress-nginx as DaemonSet, not as a service, with no admission controller
 * Allow the ingress controller to use the host network
 * For the unifi controller: specific non-HTTP(s) to be mapped so the AP discovery and adoption works

So I decided to use the _nginx ingress-controller_ instead.

Delete traefik:
```
pi@raspberrypi:~/unifi $ kubectl -n kube-system delete helmcharts.helm.cattle.io traefik
helmchart.helm.cattle.io "traefik" deleted
pi@raspberrypi:~/unifi $ sudo service k3s stop
```
Edit configuration:
```
pi@raspberrypi:~/unifi $ sudo vi /etc/systemd/system/k3s.service
```
At the end of the file, add _--no/deploy treafik_:
```
ExecStart=/usr/local/bin/k3s \
    server \
    --no-deploy traefik \
```
Reload service, remove traefik manifest and restart k3s service:
```
pi@raspberrypi:~/unifi $ sudo systemctl daemon-reload
pi@raspberrypi:~/unifi $ ls /var/lib/rancher/k3s/server/manifests/traefik.yaml
ls: cannot access '/var/lib/rancher/k3s/server/manifests/traefik.yaml': Permission denied
pi@raspberrypi:~/unifi $ sudo ls /var/lib/rancher/k3s/server/manifests/traefik.yaml
/var/lib/rancher/k3s/server/manifests/traefik.yaml
pi@raspberrypi:~/unifi $ sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml
pi@raspberrypi:~/unifi $ sudo service k3s start
```
Ingress can only manage http and https traffic, but the nginx ingress-controller can handle also UDP traffic.

Install nginx ingress-controller:
 * As DaemonSet
 * With no service
 * To use the host network
 * Mapping the corresponding host ports to the namespace/service:ports

Because of the UDP ports the unifi controller relies on to discover, adopt and communicate with the access points, I needed a way for this ports to be reachable for the container. A NodePort service wouldn't do, since it uses a random port on the host. I really needed to have the ports on the host bound in the same ports the application uses, so I used the port mapping, as defined in the _values_ file.

Run helm to proceed with the installation:
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace -f values.yaml
```
I used a _nginx-values.yaml_ file to pass it as parameter to helm for the nginx ingress controller installation. The file is available in my repo:
[nginx-values.yaml](https://github.com/eramons/k3s-home/blob/main/cluster/nginx-values.yaml)

### 2. Unifi 

The unifi controller can be downloaded and installed by end users to control the unifi access points. There are several docker images publicily available. 

#### 2.1. K8s resources 

The K8s resources are available in my github repo:
 * deployment: [unifi-deployment.yaml](https://github.com/eramons/k3s-home/blob/main/unifi/unifi-deployment.yaml)
 * persistent volume claim [unifi-pvc.yaml](https://github.com/eramons/k3s-home/blob/main/unifi/unifi-pvc.yaml)
 * service: [unifi-service.yaml](https://github.com/eramons/k3s-home/blob/main/unifi/unifi-service.yaml)
 * ingress: [unifi-ingress.yaml.sample](https://github.com/eramons/k3s-home/blob/main/unifi/ingress.yaml.sample)

Some points worth to mention regarding the manifests:
 * The deployment pulls the image provided by _linuxserver.io_, available in _dockerhub_ 
 * For storage, I use the _local provisioner_ provided by K3s: 
```
_TODO: kubectl get storageclass_
```

 * I would be mounting the _/config_ directory in the container on a local path on the raspberry pi. For that, I defined the _volumeMount_ in the deployment manifest and a _persistent volume claim_ which uses the _local-path_ storage class  
 * The service defines all TCP and UDP port used by the unifi controller, to communicate either with the end user or with the access points 
 * The ingress uses the _certmanager_ and _clusterissuer_ to request and retrieve a server SSL certificate to protect the dashboard website. In the ingress manifest provided, _myhost.mydomain.com_ must be replaced as needed, and the file must be renamed to _unifi-ingress.yaml_

With all manifests ready, use _kustomize_ to deploy the unifi K8s resources:
```
pi@raspberrypi:~/unifi $ kubectl apply -k .
```
The _kustomization.yaml_ file for the unifi K8s resources is available in my repo:
[unifi/kustomization.yaml](https://github.com/eramons/k3s-home/blob/main/unifi/kustomization.yaml)

#### 2.2. Move access points to the new controller 

Once everything is up und running, the new controller should be reachable under _myhost.mydomain.com_.

To move from the previous unifi controller running on the docker application on my synology, I did the following on the console of the old controller:
 * Go to the _export site_
 * Download the backup file
 * _Forget device_ for each of the two access points

Then in the console of the new controller:
 * Restore the backup. When accessing the new controller, the possibility to directly restore a backup is offered at the beginning. 
 * _Adopt_ the access points

After the initial setup, the credentials to access the new controller will be the same as for the old one.

Only one thing left: shut down the unifi container running on the synology. 

And that's it: that's how the unifi controller became the first application in the home lightweight kubernetes cluster. Other application might now follow :)  

#### 2.3. Logging

There are two types of logs I like to mention:
 * The logs of the controller itself
 * The logs of the access points

The logs of the controller can be found in the mount point of the _config_ directory, in my case available in a directory on the raspberry pi itself:
```
_TODO_
```

The logs of the access points are sent via syslog to a syslog server. You can either configure your own, or you can just use the controller, which acts itself as a syslog server. In that case, the logs will be available under (...)
```
_TODO_
```

With this in mind, the next project would be to centralize the logs collection and visualize them. But that's enough material for a dedicated blog post. 

### Links:

https://docs.k3s.io/upgrades/manual

https://letsencrypt.org/

https://helm.sh

https://helm.sh/docs/intro/install

https://cert-manager.io/docs/

https://cert-manager.io/docs/installation/helm

https://cert-manager.io/docs/configuration/acme/

https://keepforyourself.com/kubernetes/disable-traefik-from-k3s/

https://kubernetes.github.io/ingress-nginx/deploy/

https://hub.docker.com/r/linuxserver/unifi-controller

https://docs.k3s.io/storage
