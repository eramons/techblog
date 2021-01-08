+++
math = true
date = "2021-01-07T09:00:00+02:00"
title = "WIP Nextcloud self-hosting on K8s"
tags = ["nextcloud", "/e", "kubernetes"]
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Self-host Nextcloud on Kubernetes and use it as server-side for the /e operating system_

Nextcloud offers an on-premises content collaboration platform, which is open-source and free. The Nextcloud server is written in the PHP and JavaScript scripting languages.

The /e/ ROM is a fork of Android (more precisely from LineageOS). See previous post to see how to install the OS on a LG G3 and to see my efforts to self-host the /e server beta version.

The /e self-hosting exercise turned to be quite cumbersome due to the installation script of the beta version, which is one of these do-everything-in-one-script which is difficult to troubleshoot und makes maintenance and updates complicated. However, the /e server-side is just a composition of services:

 * Nextcloud, as file synchronization software
 * Postfix, to host the own mail server
 * OnlyOffice, to allow editing of documents 

Since I actually only wanted to have Nextcloud - being interested on photos and files synchronization only - I deciced to try to install Nextcloud directly and see if I could use it as the server side for the /e operating system on the phone. The nice guys from /e advised me to do so, too.

As a hosting environment, as usual I aimed to use Kubernetes. This time - instead of using my K8s or my K3s cluster on premises - I wanted to try the Digital Ocean K8s services. Afterwards I could move it to my home cluster to use it productively to sync my files and photos. 

_NOTE: another reason was that by the time of this writting I was on vacation with no access to my home network ;)_

## Pre-conditions

 * A DigitalOcean account to host the K8s cluster
 * A client device to test the installation
 * A hosted public domain (_mydomain.com_)
 * A DNS provider to register the necessary DNS entries 

I had a DigitalOcean account and I was still enjoying the free credit I got when signing up for the first time. The client device will be the LG G3 where /e already was running. My public domain was hosted by GoDaddy and as DNS provider I will use Cloudfare as usual.

## Milestones

 1. Set up a single-node K8s cluster on DigitalOcean 
 2. Identify software components and write K8s manifests 
 3. Set up DNS 
 4. Certmanager and ingress controller
 5. Deployment
 6. Connect /e client to nextcloud

## 1. Set up a K8s cluster 

Setting up a K8s cluster on DigitalOcean was easy:
 
 * I selected the latest K8s version, which was the default
 * I selected the datacenter closest to my current location
 * I selected one single _basic nodes_ with _2.5 GB RAM usable (4 GB Total)/2 vCPUs_ 
 * As cluster name, I typed _eramonk8s_

## 2. Install and configure _kubectl_ and _doctl_

As I waited for the provisioning of the cluster, I installed version 1.19.3 of _kubectl_:

eramon@pacharan:~/dev$ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.3/bin/linux/amd64/kubectl

_NOTE: the kubectl version must match the K8s version of the cluster_

eramon@pacharan:~/dev$ chmod +x ./kubectl

eramon@pacharan:~/dev$ sudo mv ./kubectl /usr/local/bin/kubectl

eramon@pacharan:~/dev$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}

I also downloaded the _doctl_ tool as recommended by the DigitalOcean wizard:

eramon@pacharan:~/dev$ wget https://github.com/digitalocean/doctl/releases/download/v1.54.0/doctl-1.54.0-linux-amd64.tar.gz

eramon@pacharan:~/dev$ tar xvzf doctl-1.54.0-linux-amd64.tar.gz

eramon@pacharan:~/dev$ sudo mv doctl /usr/local/bin/

I used _doctl_ for automated certificate management and download of the configuration file. Following the instructions, I first ran:

eramon@pacharan:~/dev$ doctl auth init
Please authenticate doctl for use with your DigitalOcean account. You can generate a token in the control panel at https://cloud.digitalocean.com/account/api/tokens

Enter your access token: 
Validating token... OK

I created the access token on the URL above and gave it the same name as the cluster. I copied it to the clipboard and pasted it on the console as instructed.

Then I used _doctl_ to generate the configuration file:

eramon@pacharan:~/dev$ doctl kubernetes cluster kubeconfig save eramonk8s
Notice: Adding cluster credentials to kubeconfig file found in "/home/eramon/.kube/config"
Notice: Setting current-context to do-fra1-eramonk8s

After this I was able to connect to my cluster via kubectl:

eramon@pacharan:~/dev$ kubectl cluster-info
Kubernetes master is running at https://73c21301-5143-42fa-ab21-1c4c54e9b1f0.k8s.ondigitalocean.com
CoreDNS is running at https://73c21301-5143-42fa-ab21-1c4c54e9b1f0.k8s.ondigitalocean.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

Continuing with the wizard on the browser, I installed the following _1-Click Apps_:
 
 * NGINX Ingress Controller
 * Kubernetes Monitoring Stack 

Over the control panel I was able to access the _Kubernets Dashboard_ to monitor the status of the cluster.

## 2. Software components and manifests

Nextcloud is built upon the following components:

 * The nextcloud webapp: PHP + Apache 
 * MariaDB for metadata storage 
 * Persistent storage to save content

### 2.1 Secrets

I started generating the secrets we'll need for the manifests later:

Install pwgen:
eramon@pacharan:~/dev/kubenextcloud$ sudo apt-get install pwgen

Use pwgen to generate a random password for mariadb:
eramon@pacharan:~/dev/kubenextcloud$ echo -n `pwgen -s -1 16`

Note the output.

To simplify, I used the same generated password for _MYSQL\_PASSWORD_ and for _MYSQL\_ROOT\_PASSWORD_. 

As _MYSQL\_DATABASE_ and _MYSQL\_USER_, I set _nextcloud_:
Use openssl to base64-encode "nextcloud": 

eramon@pacharan:~/dev/kubenextcloud$ echo -n 'nextcloud' | openssl base64
bmV4dGNsb3Vk

Write a manifest for the secrets, using the generated passwords:

apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secrets
type: Opaque
data:
  MYSQL_DATABASE: <base64-encoded "nextcloud"> 
  MYSQL_USER: <base64-encoded "nextcloud"> 
  MYSQL_PASSWORD: <base64-encoded pwgen-generated password> 
  MYSQL_ROOT_PASSWORD: <base64-encoded pwgen-generated password> 

Create a new _kustomize.yaml_ file and included the manifest for the secrets there.  

### 2.2 Persistent Volumes

To see the possibilities for the creation of a persistent volume with DigitalOcean, I got to the _Volumes_ section on the _Manage_ menu. One persistent volume claim was already created - it was automatically set for the use of the monitoring tool.   

I went to the _How to Add Block Storage Volumes to Kubernets Clusters_ tutorial. Taking the code from the example, I created a manifest for a 3GB block storage volume:

[nextcloud-pvc.yaml](https://github.com/eramons/kubenextcloud/blob/master/nextcloud-pvc.yaml)

I created an additional volume for the storage of the mariadb metadata with size 2GB:

[mariadb-pvc.yaml](https://github.com/eramons/kubenextcloud/blob/master/mariadb-pvc.yaml)

For managing all manifests and easing the deployment, I would use _kustomize_.

Add the pvc manifests to _kustomization.yaml_.

### 2.3 MariaDB

Create manifest file for the mariadb deployment:

[mariadb-deployment.yaml](https://github.com/eramons/kubenextcloud/blob/master/mariadb-deployment.yaml)

The mariadb docker image allows the creation of a database upon creation of the container. 

_When you start the mariadb image, you can adjust the configuration of the MariaDB instance by passing one or more environment variables on the docker run command line. Do note that none of the variables below will have any effect if you start the container with a data directory that already contains a database: any pre-existing database will always be left untouched on container startup._

 * MYSQL_ROOT_PASSWORD: specifies the password that will be set for the MariaDB root superuser account (mandatory)
 * MYSQL_DATABASE: allows to specify the name of a database to be created on image startup (optional)
 * MYSQL_USER, MYSQL_PASSWORD: used in conjunction to create a new user and to set that user's password (optional)

_NOTE In order for mariadb to re-create the nextcloud database, the persistent volume must be deleted. It's not enough with delete the container. The environment variables only work if the database is started for the first time._ 

Create manifest file for the mariadb service:

[mariadb-service.yaml](https://github.com/eramons/kubenextcloud/blob/master/mariadb-service.yaml)

Add the deployment and service manifest files to _kustomization.yaml_.

### 2.4 Nextcloud

Create manifest file for the nextcloud deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud 
  labels:
    app: nextcloud 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud 
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      volumes:
      - name: nextcloud-storage
        persistentVolumeClaim: 
          claimName: nextcloud-pvc
      containers:
        - image: nextcloud:apache
          name: nextcloud 
          ports:
            - containerPort: 80
          env:
            - name: MYSQL_HOST
              value: mariadb 
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  key: MYSQL_DATABASE
                  name: mariadb-secrets
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: MYSQL_PASSWORD
                  name: mariadb-secrets
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  key: MYSQL_USER
                  name: mariadb-secrets
            - name: NEXTCLOUD_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: MYSQL_PASSWORD
                  name: mariadb-secrets 
            - name: NEXTCLOUD_ADMIN_USER
              value: "admin"
            - name: NEXTCLOUD_TRUSTED_DOMAINS 
              value: _<public IP of the load balancer>_
          volumeMounts:
            - mountPath: /var/www/html
              name: nextcloud-storage

The Nextcloud image supports auto configuration via environment variables. You can preconfigure everything that is asked on the install page on first run. 

 * MYSQL_DATABASE: Name of the database using mysql / mariadb
 * MYSQL_USER: Username for the database using mysql / mariadb
 * MYSQL_PASSWORD: Password for the database user using mysql / mariadb
 * MYSQL_HOST: Hostname of the database server using mysql / mariadb

With a complete configuration by using all variables for your database type, you can additionally configure your Nextcloud instance by setting admin user and password:

 * NEXTCLOUD_ADMIN_USER: Name of the Nextcloud admin user. I used _admin_.
 * NEXTCLOUD_ADMIN_PASSWORD: Password for the Nextcloud admin user. For simplification - since this is just a experimental setup - I configured it to be the same password as the one used for the nextcloud database.

One or more trusted domains can be set through environment variable too:

 * NEXTCLOUD_TRUSTED_DOMAINS: optional space-separated list of IPs or domains. The IP of the loadbalancer must be configured here.

Create manifest file for the nextcloud service:

[nextcloud-service.yaml](https://github.com/eramons/kubenextcloud/blob/master/nextcloud-service.yaml)

Add the nextcloud deployment and service manifest files to _kustomization.yaml_.

## 2.5 Ingress

_Kubernetes Ingresses allow you to flexibly route traffic from outside your Kubernetes cluster to Services inside of your cluster. This is accomplished using Ingress Resources, which define rules for routing HTTP and HTTPS traffic to Kubernetes Services, and Ingress Controllers, which implement the rules by load balancing traffic and routing it to the appropriate backend Services._

I had already installed the nginx-ingress-controller as 1-click up when setting up the cluster at the beginning:

eramon@pacharan:~/dev/kubenextcloud$ kubectl get pods -n ingress-nginx
NAME                                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-ingress-nginx-controller-7898d5969d-sgph4   1/1     Running   0          110m

Create manifest file for ingress:

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nextcloud-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    cert-manager.io/acme-challenge-type: http01
spec:
  tls:
  - hosts: 
    - nextcloud.mydomain.com 
    secretName: nextcloud-tls 
  rules:
  - host: nextcloud.mydomain.com 
    http: 
      paths: 
        backend:
          serviceName: nextcloud
          servicePort: 80

Don't forget to include the _tls_ directive and the certbot and letsencrypt anotations for the ssl certificate to be generated for the server application.

Add the ingress manifest file to _kustomization.yaml_.

## 3. DNS

I have my domain _mydomain.com_ registered and hosted by _godaddy_. I use Cloudflare to manage the DNS configuration for my projects.

I listed the services on namespace _nginx-ingress_ to find out the public IP of the load balancer of the cluster:

eramon@pacharan:~/dev/kubenextcloud$ kubectl get service -n ingress-nginx
NAME                                               TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.245.56.207   <publicIP>   80:32519/TCP,443:30323/TCP   111m

On Cloudflare, I added a new DNS entry:

 * Type: A record
 * Name: nextcloud.mydomain.com
 * Content: _<public IP of the load balancer>_

## 4. Certmanager

Install certmanager and its custom resource definitions directly from manifest:

eramon@pacharan:~/dev/kubenextcloud$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml

Verify the installation:

eramon@pacharan:~/dev/kubenextcloud$ kubectl get pods --namespace cert-manager
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-cainjector-fc6c787db-g9d5g   1/1     Running   0          67s
cert-manager-d994d94d7-7fzkd              1/1     Running   0          67s
cert-manager-webhook-845d9df8bf-d98vl     1/1     Running   0          66s

After installing certmanager, write a manifest for the cluster issuer:

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # EMail address used for ACME registration
    email: nextcloud@mydomain.com
    # Name of a secret to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod

    # Enable HTTP01 challenge provider using nginx
    solvers:
    - http01:
        ingress:
          class: nginx

Add the cluster issuer to _kustomization.yaml_.

### 5 Deployment 

After completing all manifests and including them in _kustomization.yaml_, the file looked like this:

[kustomization.yaml](https://github.com/eramons/kubenextcloud/blob/master/kustomization.yaml)

Finally, to deploy everything:

eramon@pacharan:~/dev/kubenextcloud$ kubectl apply -k .

When accessing _nextcloud.mydomain.com_ I was greeted with the login page. I logged in as Administrator and created a user _eramon_ in the administration console, with e-mail _eramon@mydomain.com_ (the e-mail is arbitrary) and a password.

I logged out and logged in again, using the newly created user. 

### 6 Synchronize with /e ROM

On my LG G3, I added a new /e account:

 * e-mail: _eramon@mydomain.com_ (the user I created before)
 * password: the password I set before
 * server: _https://nextcloud.mydomain.com_

It worked. My /e phone was now connected to my own nextcloud installation running on K8s :) 


## Links

[DigitalOcean](https://www.digitalocean.com)

[Nextcloud wiki](https://en.wikipedia.org/wiki/Nextcloud)

[Nextcloud](https://nextcloud.com)

[Doctl](https://github.com/digitalocean/doctl/releases)

[How to connect to a DO K8s Cluster](https://www.digitalocean.com/docs/kubernetes/how-to/connect-to-cluster)

[How to Add Block Storage Volumes to K8s clusters](https://www.digitalocean.com/docs/kubernetes/how-to/add-volumes)

[Cloudflare](https://www.cloudflare.com)

[Ingress and Certmanager on DO K8s](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes)
