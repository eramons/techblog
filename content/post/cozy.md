+++
math = true
date = "2020-05-20T17:00:00+02:00"
title = "Self-hosted cozy on K8s"
tags = []
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Deploy cozy on my homemade kubernetes cluster_

The k8s homemade cluster on baremetal was meant to be just an exercise and not a permanent setup. Still I needed to deploy something on it in order to properly try it out :)

So I decided to try out Cozy Cloud deploying the software and its dependencies as containers in my K8s cluster.

Cozy is a personal, free and self-hostable cloud platform, written in Go. The platform aims at simplifying the use of a personal cloud and at allowing the users to take back ownership of their privacy.  

__Milestones:__
 
1. Applications
2. Networking
3. Docker images
4. Cozy configuration
5. k8s configuration
6. Create cozy instance


## 1. Applications 

According to the developer documentation, there are following ways of installing and self-hosting cozy:

* using cozy-customized debian packages (non-debian official) following the self-hosting guide
* self build the application
* run a docker developer cozy image, which all dependencies and configuration included 

The "official" way seemed to use the debian packages. This way does not only install cozy-stack and their dependencies, but also a nginx reverse proxy and an application called cozy-coclyco which is meant to manage cozy instances and SSL certificates. 

I didn't need coclyco, since my cluster uses cert-manager and nginx-ingress for SSL termination and issuance and renewal of TLS server certificates.

I figured out that the only part of cozy I needed was actually the cozy-stack binary application.

Basing on this premise, I identified the applications and dependencies to be considered:

### 1.1. Applications
* __cozy-stack__: the core server of the cozy platform, which consists of a single process
* __couchdb__: database for storing the cozy application data

### 1.2. Dependencies (cozy-stack)
* __mailhog__: cozy-stack needs a smtp server in order to work. _Mailhog_ is an e-mail testing tool for developers. I intended to use mailhog instead of configuring a smtp server.
* __imagemagick__: image manipulation program binaries

## 2. DNS and reverse-proxy

As a security measure, Cozy needs to be served over HTTPS, which means it needs a reverse proxy in front of it. 

Cozy needs a full domain name for the instance (something like <instance>.mydomain.net) and use one domain name per application, in the form of <app>.<instance>.mydomain.net

Thus, the setup of the domain zone has to look like this:
```
   <instance> 1h IN A <my external IP> 
   *.<instance> 1h IN CNAME <instance>
```

Currently the list of apps is: home, settings, drive, photos, onboarding.

A wildcard SSL certificate covering *.cozy.mydomain.net and cozy.mydomain.net or a certificate for cozy.mydomain.net with apps domains added as Subject Alternative Name.

* I used Clodflare for the DNS configuration. 
* In order to have proper DNS resolution inside of the home network, I set up an own DNS server. 

## 3. Docker 

### 3.1 Available Docker Images

From the applications identified in the last section:

* couchdb and mailhog have official docker images available
* imagemagick is available as debian package and should be included on the cozy-stack docker image
* for cozy-stack I decided to write my own Dockerfile 

### 3.2 Docker image for cozy-stack

The cozy-stack is a (statically-linked?) go binary. Two options for the new image:

   a)   Build the go binary inside the container
   b)   Pull the binary directly from github inside the container

For now, I'm going for the second option. The Dockerfile looks like follows:

```
https://github.com/eramons/kubecozy/blob/master/docker/Dockerfile
```
Let's take a closer look at the Dockerfile:

* The basis image is debian (bullseye)
* _Imagemagick_ is installed as a dependency
* I created a _cozy_ user for running cozy-stack
* The workdir is the home directory of the _cozy_ user
* The application will be executed as _cozy_ user (NOT as root)

Once ready, I build the image and push it to dockerhub:
```
docker login
```
```
docker image build -t eramon/cozy-stack .
```
```
docker push eramon/cozy-stack
```

## 2. Cozy configuration

The _cozy-stack_ binary and its command _serve_ allow to pass flags as configuration options. As an alternative, a _cozy.yaml_ configuration file can be placed under _/etc/cozy_. 

I prefered to use a config file in order to keep the configuration separated from the container image. 

My _cozy.yaml_ config file looks like follows:
```
https://github.com/eramons/kubecozy/blob/master/docker/Dockerfile
```

Let's go over the most important configuration settings:

### Service hosts & port
```
host: 0.0.0.0 
port: 8080
```
The bind address and the port where cozy-stack will listen for connections. Important to use 0.0.0.0 and NOT localhost, otherwise the application will only bind to 127.0.0.1. 

### Admin interface 
```
admin:
  host: localhost 
  port: 6060
  secret_filename: cozy-admin-passphrase
```
The admin interface to send requests to cozy-stack. The cozy admin password is stored encrypted in the filesystem. It must be generated beforehand using the cozy-stack binary (see K8s configuration: secrets).

### Database
```
couchdb:
  url: http://couchdb:5984
```
The Couchdb stores the cozy application data.

_Note: in kubernetes, an image name can be used as a resolvable hostname inside the cluster network._

### Storage
```
fs:
  url: file://localhost/var/lib/cozy
```
Filesystem path for the storage of user data: photos, documents, etc. 

### Vault
```
vault:
  credentials_encryptor_key: /etc/cozy/vault.enc
  credentials_decryptor_key: /etc/cozy/vault.dec
```
Cozy encrypts user credentials. The genertion of the encryption and decryption keys must be done manually before the cozy-stack is started (see K8s configuration: secrets).

### SMTP Server
```
mail:
  host: mailhog 
  port: 1025 
  disable_tls: true 
  skip_certificate_validation: true 
```
Apparently having a working SMTP server up und running is a requirement in order for cozy-stack to work. I wasn't actually interested in my cozy writting me e-mails. I found out the cozy development image uses _Mailhog_. The e-mails can be viewed on the browser but are not actually sent. So I configured mailog as the smtp server. 

_Note: mailhog does not allow to configure a username and a password. Otherwise, it fails with an "unencrypted connection" error_


## 4. k8s configuration

Now that I had the docker images an the cozy-stack configuration file prepared, the next step was to deploy the applications in my home-made K8s cluster.

Pre-conditions:

* To have kubectl installed on my laptop and the credentials to the k8s cluster stored under .kube/config on the home directory
* To have helm installed on my laptop
* To have cert-manager deployed on the cluster
* To have nfs-common installed on all worker nodes of the cluster

_Note: I addressed these topics already on my previous post: Home-made Kubernetes Cluster_
 
For deploying cozy in k8s, I wrote yaml manifests for:

* The services
* The secrets
* The configMap
* The ingress
* The cluster issuer

All manifest files available here:
```
https://github.com/eramons/kubecozy
```

### 4.1 Services

There are three services:

1. The couchdb database service
2. The cozy-stack service 
3. The mailhog smtp-server

__couchdb-service.yaml__:
```
apiVersion: v1
kind: Service
metadata:
  name: couchdb
  labels:
    run: couchdb
spec:
  ports:
  - port: 5984
    protocol: TCP
  selector:
    app: couchdb

```

__cozy-stack-service.yaml__:
```
apiVersion: v1
kind: Service
metadata:
  name: cozy-stack
spec:
  selector:
    app: cozy-stack
  ports:
  - port: 8080
    protocol: TCP
```

__mailhog-service.yaml__:
```
apiVersion: v1
kind: Service
metadata:
  name: mailhog 
  labels:
    run: mailhog 
spec:
  ports:
  - port: 1025 
    name: smtp-server 
    protocol: TCP
  - port: 8025
    name: http-server
    protocol: TCP
  selector:
    app: mailhog 
```

### 4.2 Secrets

#### 4.2.1 admin passphrase

I needed the cozy-stack binary for generate the admin password. Since my cozy-stack wasn't up and running yet, I started a standalone instance under docker in order to be able to use the cozy-stack binary:
```
docker run --rm -it -p 8080:8080 -v "$(pwd)/build":/data/cozy-app/evita cozy/cozy-app-dev
docker ps
```
I found out the container id and logged into the running container:
```
eramon@caipirinha:~/dev/cozy/docker$ docker exec -it 0b1400039866 /bin/bash
root@0b1400039866:/# 
```

Then I generated the password and copied the file outside the container:
```
root@0b1400039866:/# cozy-stack config passwd cozy-admin-passphrase
Hashed passphrase will be written in /cozy-admin-passphrase
Passphrase: 
Confirmation: 
root@0b1400039866:/# 
exit
eramon@caipirinha:~/dev/cozy/docker$ docker cp 0b1400039866:/cozy-admin-passphrase ./../../kubernetes/cozy-admin-passphrase
```

I manually generated a K8s secret for the password:
```
eramon@caipirinha:~/dev/kubernetes$ kubectl create secret generic cozy-admin-passphrase-secret --from-file=cozy-admin-passphrase
secret/cozy-admin-passphrase-secret created

```
This secret will be mounted afterwards in the corresponding location on the cozy-stack pod, via the deployment manifest.

#### 4.2.2 vault keys

_TODO Open - I need the cozy binary_

#### 4.2.3 couchdb credentials

_TODO_ Not sure. I could create them and include them either as ENV or directly on ...

### 4.3 ConfigMap

As mentioned in the _Configuration_ section, I decided to use a config file for cozy-stack rather than having flags passed as arguments to cozy-stack. The file cozy.yaml must be included in a ConfigMap and mounted under /etc/cozy/cozy.yaml in the cozy-stack container.

To generate the config map, I directly include it in the resource list of kustomization.yaml:
```
# ConfigMaps
configMapGenerator:
- name: cozy-config-map
  files:
  - config/cozy.yaml
```

### 4.4 Persistent Volume 

Preparations:
* I set up a NFS share folder on my synology NAS
* I installed nfs-common on my worker's node

#### Set up NFS Share on Synology NAS

Setting up the NFS share on the Synology NAS was quite straightforward. 

_TODO Screenshot_

#### Persistent Volumes

After the NFS share was available on my NAS, I wrote the manifests for the persistent volumes, which reference the NFS serverIP and the path to the share:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cozy-nfs
spec:
  capacity:
    storage: 10Gi
  storageClassName: standard
  accessModes:
  - ReadWriteMany
  nfs:
    server: 192.168.1.xyz
    path: /volume1/cozy
``` 

Same for the couchdb data:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cozy-nfs
spec:
  capacity:
    storage: 10Gi
  storageClassName: standard
  accessModes:
  - ReadWriteMany
  nfs:
    server: 192.168.1.xyz
    path: /volume1/cozy/storage
```

#### Persistent Volume Claims for storage and data

To bind the persistent volumes to the pods, I needed to create the Persistent Volume Claims.

__cozy-stack-pvc.yaml__ (storage i.e user data):
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cozy-nfs 
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

__couchdb-pvc.yaml__ (application data):
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: couchdb-nfs
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

In order for cozy-stack and couchdb to use this persistent volume to store its data, the corresponding volume mounts must be configured on their respective deployment manifests.

### 4.4 Deployments

Finally, we come to the manifests for the deployments of couchdb, cozy-stack and mailhog.

#### 4.4.1 couchdb
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: couchdb
  labels:
    app: couchdb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: couchdb
  template:
    metadata:
      labels:
        app: couchdb
    spec:
      volumes:
      - name: couchdb-data
        persistentVolumeClaim:
          claimName: couchdb-nfs
      containers:
      - name: couchdb
        image: apache/couchdb:2.3
        ports:
          - containerPort: 5984
        volumeMounts:
        - name: couchdb-data
          mountPath: /opt/couchdb/data
```

The pvc is mounted under /opt/couchdb/data.

#### 4.4.2 cozy-stack
```
https://github.com/eramons/kubecozy/blob/master/cozy-stack-deployment.yaml
```
_TODO Not sure if I want to include here the code of all manifests or just the links. 

Taking a closer look:

* The configMap containing the cozy.yaml is mounted under /etc/cozy/cozy.yaml
* The persistent volume (nfs) is mounted under /var/lib/cozy
* Since the docker image is NOT ran as root, I needed an init container to mount the persistent volume and set the permissions
* The secret containing the admin-passphrase is mounted under /etc/cozy/cozy-admin-passphrase

#### 4.3 mailhog
```
https://github.com/eramons/kubecozy/blob/master/mailhog-deployment.yaml
```

### 4.5 ClusterIssuer

Issuers (and ClusterIssuers) represent a certificate authority from which signed x509 certificates can be obtained, such as Let’s Encrypt. I needed one ClusterIssuer in order to generate certificates for my cluster with the cert-manager. 

DNS01 solver with acme version 2 is mandatory for issuance of wildcard certificates. I needed a wildcard certificate to cover all subdomains use by the cozy applications, so I used a DNS01 challenge with an api token provided by Cloudflare.

__cozy-ingress.yaml__
```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # EMail address used for ACME registration
    email: eva.ramon@swisscom.com
    # Name of a secret to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod

    # Enable HTTP01 challenge provider using nginx
    solvers:
    # Enable DNS01 challenge provider using Cloudflare
    - dns01:
        cloudflare:
          email: me@mydomain.net 
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

### 4.5 Ingress

The ingress must expose the cozy-stack via the nginx ingress controller installed previously.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cozy-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    cert-manager.io/acme-challenge-type: dns01
spec:
  tls:
  - hosts:
    - '*.empanadilla.net'
    - empanadilla.net
    secretName: empanadilla-tls
  rules:
  - host: '*.empanadilla.net'
    http:
      paths:
      - path: /
        backend:
          serviceName: cozy-stack
          servicePort: 8080
  - host: empanadilla.net
    http:
      paths:
      - path: /
        backend:
          serviceName: cozy-stack
          servicePort: 8080
```

* The ingress redirects all requests to *.mydomain.net and mydomain.net to the cozy-stack service listening in port 8080.
* Including two hosts in the _tls_ section results in the issuance of a certificate including both of them

_TODO CORS not finished_

### 4.6 kustomize

In order to ease re-deployments, I used _kustomize.

Kustomize introduces a template-free way to customize application configuration and it's built into the _kubectl_ command. After any change to a configuration file or manifest file, it's enough to run:
```
kubectl apply -k .
```

The _kustomization.yaml_ file includes all manifests and config files.
```
https://github.com/eramons/kubecozy/blob/master/kustomization.yaml
```
```
eramon@caipirinha:~/dev/kubernetes$ cat kustomization.yaml 
resources:

# Ingress and certmanager
- letsencrypt-prod.yaml
- ingress.yaml
- cloudflare.yaml

# Secrets
secretGenerator:
- name: cozy-admin-passphrasse-secret
  files:
  - cozy-admin-passphrase

# Deployments
- couchdb-deployment.yaml
- cozy-stack-deployment.yaml

# Services
- couchdb-service.yaml
- cozy-stack-service.yaml
```

For applying any configuration changes to the cluster, it's enough to execute:
```
kubectl apply -k .
```

## 7. Create instance

The Dockerfile includes instructions for downloading the latest cozy-stack binary, create directories, create a cozy user, set permissions, create the vault keys and finally start cozy-stack serve. The creation of the instance must be done manually once by each user and it must not be part of the docker image. 

For executing the cozy-stack binary to create my cozy instance, I logged into the running container:
```
kubectl exec -it cozy-stack-b6dbd5db8-m58dt /bin/bash
```

Then I created a new cozy-stack instance:
```
cozy@cozy-stack-b6dbd5db8-4j2fw:~$ cozy-stack instances add --passphrase cozy --apps home,drive,photos,settings cozy.mydomain.net     
Password:****
Instance created with success for domain cozy.mydomain.net
Password:****
```

Show running instances:
```
cozy@cozy-stack-b6dbd5db8-4j2fw:~$ cozy-stack instances ls
Password:****
cozy.mydomain.net  en  unlimited  onboarded  v27  cozyf55261b8a5b88690387587c7af0bb8af  cozyf55261b8a5b88690387587c7af0bb8af
```

Looking good :)

After this, I was able to connect to my cozy applications under following urls:
https://empanadilla.net/drive
https://empanadilla.net/photos
https://empanadilla.net/settings

In order to be able to access to the applications, I have to log in using the passphrase defined during the instance creation.

I tried following client applications on my devices:
- On my android mobile phone, _cozy drive_, available in the Google Playstore.
- On my debian laptop, the cozy desktop client
- The browser
```

TODO Screenshot

## Links:
CORS
Kustomize
[Kubernetes](https://kubernetes.io/docs/home)
[Archlinux](https://wiki.archlinux.org/index.php/Cozy)
[Desktop client](https://docs.cozy.io/en/howTos/sync/linux)
[Mailhog](https://github.com/mailhog/MailHog)

