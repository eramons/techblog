+++
math = true
date = "2020-05-13T10:00:00+02:00"
title = "Self-hosted cozy on k8s"
tags = []
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Deploy cozy on my home made kubernetes cluster__

The k8s home-made cluster on bare-metal was meant to be an exercise more than a permanent setup. Still I needed at least an application in order to try it out :)

Cozy is ... TODO

According to the developer documentation, there are following "official" ways for hosting cozy:
- non-official debian packages
- ?
- self-built

__Milestones:__

1. Applications
2. Cozy configuration
3. Write docker image
4. k8s configuration 
5. Create cozy instance

## 1. Applications 

First of all, we need to identify the applications we want to deploy to the cluster.

TODO

* couchdb
* cozy-stack

## 2. Cozy configuration

* Storage
* Database
* Admin password
* Vault?

TODO Include changes to sample cozy.yaml and link to github
```
# server host - flags: --host
host: 0.0.0.0 
# server port - flags: --port -p
port: 8080

The property _host_ must be set to _cozy-stack_. If setting it to _localhost_, cozy-stack only will listen for connections in the loopback interface.

# Subdomain structure for apps: "nested" or "flat" (what is the default?)
# "nested" is well suited for self-hosting with Let's Encrypt
subdomains: nested

# vault
# TODO 

# couchdb parameters
couchdb:
  # CouchDB URL - flags: --couchdb-url (maybe better to include it in the Dockerfile and remove?)
  url: http://couchdb:5984/

```

## 3. Write docker image
Although there are some instructions and several cozy docker images available in dockerhub, there is no up-to-date cozy-stack docker image which I could directly deploy to k8s.

The cozy-stack is a (statically-linked?) Go binary (...)

* Build the go binary inside the container
* Pull the binary directly from github inside the container

I wrote the Dockerfile:
```
eramon@caipirinha:~/dev/kubernetes/cozy-stack$ cat Dockerfile 
FROM debian:bullseye
WORKDIR /home/cozy 

RUN apt-get -y update && apt-get -y install wget curl imagemagick && wget -O /usr/local/bin/cozy-stack https://github.com/cozy/cozy-stack/releases/download/1.4.9/cozy-stack-linux-amd64 && chmod +x /usr/local/bin/cozy-stack

# Copy configuration file
COPY cozy.yaml /etc/cozy/cozy.yaml

# Add cozy user and directories
RUN useradd -M cozy && mkdir -p /etc/cozy && chown -hR cozy /etc/cozy && chown -hR cozy /home/cozy && mkdir -p /var/lib/cozy && chown -hR cozy /var/lib/cozy

# Change to non-root user
USER cozy

# Environment
ENV COUCHDB_USER admin
ENV COUCHDB_PASSWORD ""

CMD ["/usr/local/bin/cozy-stack","serve"]
```

Let's take a closer look at the Dockerfile:

* The basis image is debian (bullseye)
* I created a _cozy_ user for running cozy-stack
* The workdir is the home directory of the _cozy_ user
* The application will be executed as _cozy_ user (NOT as root)
* The configuration file is installed under /etc/cozy

Once ready, I built the image and push it to dockerhub:
```
eramon@caipirinha:~/dev/kubernetes/cozy-stack$ docker login
eramon@caipirinha:~/dev/kubernetes/cozy-stack$ docker image build -t eramon/cozy-stack .
eramon@caipirinha:~/dev/kubernetes/cozy-stack$ docker push eramon/cozy-stack
```

## 4. k8s configuration

Pre-conditions:

* To have kubectl installed and the credentials to the k8s cluster stored under .kube/config on the home directory
* To have helm installed
* To have cert-manager deployed on the cluster

I addressed these topics already on my previous post _Home-made Kubernetes Cluster_
 
For deploying cozy in k8s, I wrote yaml manifests for:

* The services
* The secrets
* The ingress
* The cluster issuer

TODO Files, description and github links

### 4.1 Services

There are two services:

1. The couchdb database (...)
2. The cozy-stack (...)

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

### 4.2 Secrets

### 4.3 Deployments

### 4.4 ClusterIssuer

The ClusterIssuer (...) TODO

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
#    - http01:
#        ingress:
#          class: nginx
    # Enable DNS01 challenge provider using Cloudflare
    - dns01:
        cloudflare:
          email: eramonsalinas@gmail.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

DNS01 solver with acme version 2 is mandatory for issuance of wildcard certificates. 

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

### 4.6 kustomize

In order to ease re-deployments, I used _kustomize_ including all manifests in a _kustomization.yaml_ file:
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
cozy@cozy-stack-b6dbd5db8-4j2fw:~$ cozy-stack instances add --passphrase <passphrase> --apps drive,photos,settings empanadilla.net     
Password:****
Instance created with success for domain empanadilla.internet-box.ch
Password:****
```

Another - nicer - way is to run _cozy-stack_ directly via _kubectl_:
```
kubectl cozy-stack instances add <arguments>
```

Show running instances:
```
cozy@cozy-stack-b6dbd5db8-4j2fw:~$ cozy-stack instances ls
Password:****
empanadilla.net  en  unlimited  onboarded  v27  cozyf55261b8a5b88690387587c7af0bb8af  cozyf55261b8a5b88690387587c7af0bb8af
```

Looking good :)

After this, I was able to connect to my cozy applications under following urls:
https://empanadilla.net/drive
https://empanadilla.net/photos
https://empanadilla.net/settings

In order to be able to access to the applications, I have to log in using the passphrase defined during the instance creation.

TODO Screenshot

## Links:
https://wiki.archlinux.org/index.php/Cozy

