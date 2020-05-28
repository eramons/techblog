+++
math = true
date = "2020-05-28T17:00:00+02:00"
title = "Self-hosted cozy on K8s"
tags = []
highlight = true

[header]
  caption = ""
+++
 
__Goal:__

_Deploy cozy on my homemade kubernetes cluster_

The K8s homemade cluster on baremetal was meant to be just an exercise and not a permanent setup. Still, I needed to find something to deploy on it :)

I decided to try out Cozy Cloud deploying software and dependencies as containers in my K8s cluster.

Cozy is a personal, free and self-hostable cloud platform, written in Go.

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

The official way seemed to be to use the debian packages. This way does not only install cozy-stack and their dependencies, but an application called cozy-coclyco which manages cozy instances and SSL certificates. During the installation, the packages prompt the user to provide usernames and passwords and generate the configuration.

I didn't need coclyco, since my cluster uses cert-manager and nginx-ingress for SSL termination and issuance and renewal of TLS server certificates. I figured out that the only part of cozy I needed was actually the cozy-stack binary application.

Basing on this premise, I identified applications and dependencies to be considered:

### 1.1. Applications
* __cozy-stack__: the core server of the cozy platform, which consists of a single process
* __couchdb__: database for storing the cozy application data

### 1.2. Dependencies (cozy-stack)
* __mailhog__: cozy-stack needs a smtp server in order to work. _Mailhog_ is an e-mail testing tool for developers. I intended to use mailhog instead of configuring a smtp server.
* __imagemagick__: image manipulation program binaries

## 2. DNS and reverse-proxy

As a security measure, Cozy needs to be served over HTTPS, which means it needs a reverse proxy in front of it. 

Cozy needs a full domain name for the instance (something like instance.example.com) and use one domain name per application, in the form of app.instance.example.com

Thus, the setup of the domain zone has to look like this:
```
   <instance> 1h IN A <my external IP> 
   *.<instance> 1h IN CNAME <instance>
```

Currently the list of apps is: home, settings, drive, photos, onboarding.

So a wildcard SSL certificate covering *.cozy.example.com and cozy.example.com is needed, or alternatively a certificate for cozy.example.com with the app domains added as Subject Alternative Name.

I used cloudflare for my DNS configuration.

It remains the issue of the internal hostname resolution. Most provider's internet boxes are not able to properly route requests to the own external IP address from inside the internal network. In order to have proper DNS resolution inside of the home network, I set up my small own DNS server. 

_TODO Coming soon: DNS Configuration for bare-metal K8s_

## 3. Docker 

### 3.1 Available Docker Images

From the applications identified in the last section:

* __couchdb__ and __mailhog__ have official docker images available
* __imagemagick__ is available as debian package (and should be included on the cozy-stack docker image)
* for __cozy-stack__ I decided to create my own docker image

### 3.2 Docker image for cozy-stack

The cozy-stack is a go binary. There was two options for writting a docker image:

   a)   Build the go binary inside the container

   b)   Pull the binary directly from github inside the container

I went for the second option. The Dockerfile looks like follows:
[Dockerfile](https://github.com/eramons/kubecozy/blob/master/docker/Dockerfile)


Let's take a closer look at the Dockerfile:

* The basis image is debian (bullseye)
* _Imagemagick_ is installed as a dependency
* A _cozy_ user is created
* The _workdir_ is the home directory of the _cozy_ user
* The application will be executed as _cozy_ user (and therefore NOT as root)

Once ready, I build the image and push it to dockerhub:
```
docker login
docker image build -t eramon/cozy-stack .
docker push eramon/cozy-stack
```

## 2. Cozy configuration

The _cozy-stack_ binary and its command _serve_ allow to pass flags as configuration options. As an alternative, a _cozy.yaml_ configuration file can be placed under _/etc/cozy_. 

I prefered to use a config file in order to keep the configuration separated from the container image. 

See my [cozy.yaml](https://github.com/eramons/kubecozy/blob/master/config/cozy.yaml)

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
Filesystem path for the storage of user data: photos, documents, etc. Later we'll see on this path a nfs share folder will be mounted. 

### Vault
```
vault:
  credentials_encryptor_key: /etc/cozy/keys/vault.enc
  credentials_decryptor_key: /etc/cozy/keys/vault.dec
```
Cozy encrypts user credentials. The generation of the encryption and decryption keys must be done before the cozy-stack is started (see K8s configuration: secrets).

### SMTP Server
```
mail:
  host: mailhog 
  port: 1025 
  disable_tls: true 
  skip_certificate_validation: true 
```
Apparently having a working SMTP server up und running is a requirement for cozy-stack. I wasn't actually interested in my cozy writting me e-mails. In the cozy documentation, I read that the cozy development image use _Mailhog_. The e-mails can be viewed on the browser but are not actually sent. I liked this approach, so I did the same. 

_Note: mailhog does not allow to configure a username and a password. Otherwise, it fails with an "unencrypted connection" error_


## 4. K8s configuration

Now that I had the docker images an the cozy-stack configuration file prepared, the next step was to deploy the applications in my home-made K8s cluster.

Pre-conditions:

* To have __kubectl__ installed on my laptop and the credentials to the k8s cluster stored under _.kube/config_ on the home directory
* To have __helm__ installed on my laptop
* To have __cert-manager__ deployed on the cluster
* To have __nfs-common__ installed on all worker nodes of the cluster

I addressed these topics already on my previous post: 
[Home-made K8s cluster]({{< ref "post/kubernetes_cluster" >}})
 
For deploying cozy in K8s, I wrote yaml manifests for:

* The services
* The secrets
* The config map
* The persistent volume claims
* The cluster issuer
* The ingress

All yaml files are available under my [kubecozy](https://github.com/eramons/kubecozy).

### 4.1 Services

A K8s __Service__ is an abstract way to expose an application running on a set of pods as a network service.

I defined four services:

1. The cozy-stack service: [cozy-stack-service.yaml](https://github.com/eramons/kubecozy/blob/master/cozy-stack-service.yaml)
2. The couchdb service: [couchdb-service.yaml](https://github.com/eramons/kubecozy/blob/master/couchdb-service.yaml)
3. The smtp-server service: [smtp-service.yaml](https://github.com/eramons/kubecozy/blob/master/smtp-service.yaml)
4. The mailhog webserver service: [mailhog-service.yaml](https://github.com/eramons/kubecozy/blob/master/mailhog-service.yaml)

### 4.2 Secrets

Kubernetes __Secrets__ let you store and manage sensitive information, such as passwords, tokens and keys.

#### 4.2.1 admin passphrase

The _cozy-stack_ binary is necessary for generate the admin password. Since my cozy wasn't up and running yet, I started a standalone instance under docker in order to be able to use the binary to generate the password:
```
docker run --rm -it -p 8080:8080 -v "$(pwd)/build":/data/cozy-app/test cozy/cozy-app-dev
```
With _docker ps_, I found out the container id and logged into the running container:
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
This secret will be mounted afterwards in the expected location on the _cozy-stack_ pod, via the deployment manifest.

#### 4.2.2 vault keys

Same as for the admin passphrase, I needed to generate the vault keys beforehand and include them in a secret:
```
eramon@caipirinha:~/dev/cozy/docker$ docker exec -it 0b1400039866 /bin/bash
root@0b1400039866:/# cozy-stack config gen-keys /etc/vault
keyfiles written in:
  /etc/vault.enc
  /etc/vault.dec
root@0b1400039866:/# exit
eramon@caipirinha:~/dev/cozy/docker$ docker cp 0b1400039866:/etc/vault.enc ../../kubernetes/vault.enc
eramon@caipirinha:~/dev/cozy/docker$ docker cp 0b1400039866:/etc/vault.dec ../../kubernetes/vault.dec
```

Generate the K8s secret for the vault keys:
```
eramon@caipirinha:~/dev/kubernetes$ kubectl create secret generic cozy-vault-secret --from-file=vault.enc --from-file=vault.dec
secret/cozy-vault-secret created
```
This secret will be mounted afterwards in the configured location on the _cozy-stack_ pod, via the deployment manifest.

_Note: another (better) way would be to use an init container for generating and mount the vault keys. Contrary to the admin passphrase, the vault keys don't need to prompt the user for a passphrase._

#### 4.2.3 couchdb credentials

Generate a secret for the couchdb username and password and pass it as environment variable to the deployment.

_TODO_

### 4.3 ConfigMap

__ConfigMaps__ allow to decouple configuration artifacts from image content to keep containerized applications portable. 

As mentioned above, I decided to use a config file for rather than having flags passed as arguments. The file _cozy.yaml_ would be included in a config map and mounted under _/etc/cozy_ in the _cozy-stack_ container.

To generate the config map, I directly include it in the resource list of kustomization.yaml:
```
# ConfigMaps
configMapGenerator:
- name: cozy-config-map
  files:
  - config/cozy.yaml
```

### 4.4 Persistent Volume Claims

A __PersistentVolume (PV)__ is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned.

A __PersistentVolumeClaim (PVC)__ is a request for storage by a user.

To store user and application data, we need to use persistent volumes and bind them to the applications through persistent volume claims and volume mounts.

_Pre-condition:_ the cluster has available persistent volumes 

I created two PVCs:

1. PVC for couchdb (application data): [couchdb-pvc.yaml](https://github.com/eramons/kubecozy/blob/master/couchdb-pvc.yaml)
2. PVC for cozy-stack (user data): [cozy-stack-pvc.yaml](https://github.com/eramons/kubecozy/blob/master/cozy-stack-pvc.yaml)

The volume mounts must be configured later on the corresponding deployment manifests.

### 4.4 Deployments

Finally, we come to the manifests for the deployments of couchdb, cozy-stack and mailhog.

A __Deployment__ provides declarative updates for Pods and ReplicaSets.

#### 4.4.1 couchdb

Deployment for couchdb (application data): [couchdb-deployment.yaml](https://github.com/eramons/kubecozy/blob/master/couchdb-deployment.yaml)

* The pvc we created before is mounted under _/opt/couchdb/data_.

#### 4.4.2 cozy-stack

Deployment for cozy-stack (user data): [cozy-stack-deployment.yaml](https://github.com/eramons/kubecozy/blob/master/cozy-stack-deployment.yaml)

Taking a closer look at the file we can see:

* The configMap containing _cozy.yaml_ is mounted under _/etc/cozy/cozy.yaml_
* The persistent volume (nfs) is mounted under _/var/lib/cozy_
* Since the docker image is NOT ran as root, I needed an init container to mount the persistent volume and set the permissions
* The secret containing the admin-passphrase is mounted under _/etc/cozy/cozy-admin-passphrase_
* The secret containing the vault keys is mounted under _/etc/cozy/vault.enc_ and _/etc/cozy/vault.dec_

#### 4.4.3 mailhog

Deployment for mailhog (smtp server): [mailhog-deployment.yaml](https://github.com/eramons/kubecozy/blob/master/mailhog-deployment.yaml)

### 4.5 ClusterIssuer

__Issuers__ (and __ClusterIssuers__) represent a certificate authority from which signed x509 certificates can be obtained, such as Let’s Encrypt. I needed one ClusterIssuer in order to generate certificates for my cluster with the cert-manager. 

A DNS01 solver with acme version 2 is mandatory for issuance of wildcard certificates. I needed a wildcard certificate to cover all subdomains use by the cozy applications, so I used a DNS01 challenge with an api token provided by Cloudflare.

ClusterIssuer: [cozy-stack-deployment.yaml](https://github.com/eramons/kubecozy/blob/master/cozy-stack-deployment.yaml)

### 4.5 Ingress

An __Ingress__ manages external access to the services in a cluster, typically HTTP.

The cozy-ingress will expose our services via the nginx ingres controller installed previously:
[cozy-ingress-example.yaml](https://github.com/eramons/kubecozy/blob/master/cozy-ingress-example.yaml)

The ingress must expose following services, via the nginx ingress controller installed previously:

* Forward all requests for *.cozy.example.com (home, drive, settings, photos) to the cozy-stack service
* Forward all requests for cozy.example.com to the cozy-stack service 
* Expose the mailhog web interface, to see the e-mails generated by cozy

The TLS certificate is generated and managed by letsencrypt. For that:

* We include the cluster-issuer annotation, referencing our clusterissuer for let's encrypt
* We set the challenge type annotation to dns01, since this is the mechanism we need to prove full control of our domain cozy.example.com
* We say a secret cozy-tls must be generated for both the main domain and the wildcard one. These will result on a wildcard certificate with cozy.example.com as Subject Alternative Name 

Following annotations for the nginx-ingress controller must be included:

* _enable-cors_, in order to disable CORS in nginx. The application already handles correctly the Access-Control-Allow-Origin headers
* _proxy-body-size_, in order to set the _client_max_body_size_ parameter in the nginx configuration. I set it to 1G, according to the example in [nginx.conf](https://github.com/cozy/cozy-coclyco/blob/master/nginx.conf)
* _proxy-connect-_, _-read-_, and _-send-timeout_. The nginx-ingress controller supports websockets out of the box, however the default values of these attributes is set by default to 60s. That's not enough to backup photos from the mobile phone to the cozy drive. I set it to 2 hours (5200s).

### 4.6 kustomize

In order to ease re-deployments after configuration changes, I used _kustomize_.

__Kustomize__ introduces a template-free way to customize application configuration and it's built into the _kubectl_ command. 

After any change to a configuration file or manifest file, it's enough to run:
```
kubectl apply -k .
```

The [kustomization.yaml](https://github.com/eramons/kubecozy/blob/master/kustomization.yaml) file includes all manifests and config files:

For applying any configuration changes to the cluster, it's enough to execute:
```
kubectl apply -k .
```

## 7. Create instance

The Dockerfile includes commands for downloading the latest cozy-stack binary, create directories, create a cozy user, set permissions and finally start _cozy-stack serve_. The creation of the instances must be done manually after deployment, using the binary. The creation of the instances is not part of the image build. 

For executing the cozy-stack binary to create my cozy instance, I logged into the running container:
```
kubectl exec -it cozy-stack-b6dbd5db8-m58dt /bin/bash
```

Then I created a new cozy-stack instance:
```
cozy@cozy-stack-b6dbd5db8-4j2fw:~$ cozy-stack instances add --passphrase cozy --apps home,drive,photos,settings cozy.example.com     
Password:****
Instance created with success for domain cozy.example.com
Password:****
```

_NOTE: another way:_
```
kubectl exec -it <pod> cozy-stack instances add <arguments>
```
Show running instances:
```
cozy@cozy-stack-b6dbd5db8-4j2fw:~$ cozy-stack instances ls
Password:****
cozy.example.com  en  unlimited  onboarded  v27  cozyf55261b8a5b88690387587c7af0bb8af  cozyf55261b8a5b88690387587c7af0bb8af
```

Looking good :)

After this, I was able to connect to my cozy applications under following urls:

* https://home.cozy.example.com
* https://drive.cozy.example.com
* https://photos.cozy.example.com
* https://settings.cozy.example.com

To access the applications, I have to log in using the passphrase defined during the instance creation.

__Client applications:__

* On my android mobile phone, _cozy drive_, available in the Google Playstore.
* On my debian laptop, the cozy desktop client
* Directly access cozy applications from the browser 

## Links:

 [Cozy](https://cozy.io/)

 [Self-host in Debian](https://docs.cozy.io/en/tutorials/selfhost-debian)

 [Kubernetes](https://kubernetes.io/docs/home)

 [nginx-ingress annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/)

 [cert-manager](https://cert-manager.io/docs/configuration/acme/dns01/)
 
 [Kustomize](https://kustomize.io/)

 [Mailhog](https://github.com/mailhog/MailHog)

 [Archlinux](https://wiki.archlinux.org/index.php/Cozy)

 [Desktop client](https://docs.cozy.io/en/howTos/sync/linux)

