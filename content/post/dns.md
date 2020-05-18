+++
math = true
date = "2020-05-19T10:00:00+02:00"
title = "DNS Configuration for K8S bare-metal cluster"
tags = []
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Configure DNS, so applications running on the K8S cluster are reachable from the internet and TLS-protected__

Setting up a home made kubernetes cluster is quite straightforward. However, for deploying applications or services accessible from the internet, a home network with the standard provider's internet box is way too limited. 

In my case, I just wanted to deploy a web application (cozy) on my cluster. In order to have this working, I needed the following: 
* A FQDN or better a *.mydomain.net resolvable from the internet
* A wildcard SSL certificate for *.mydomain.net
* A port forwarding rule for the HTTP and HTTPS ports to the K8S cluster
* A fix IP or a DynDNS service identifying the host IP where the application is running
* An own DNS server with a static host or an entry pointing to the internal IP of the FQDN 
* For certmanager and Let's Encrypt, a DNS provider supporting the DNS01 solver mechanism 

## 1. Ubiquiti EdgeRouter X (SFP) as DNS server

The easiest solution for all this mess would have been to replace the provider's internet box through a router of my own. I had a EdgeRouter X SFP I once bought but never had time to set up lying around. However, short ago my internet provider replaced the internet box through a new one, featuring 10 GB internet. (...)

TODO

### 1.1 Factory reset

### 1.2 Configure switch interface

### 1.3 DNS configuration

### 1.4 Add static host mapping

## 2. Internet Box

### 2.1. DNS Server Configuration

After I had my DNS server up und running, the last thing was to change the DNS server configuration on my provider's internet box to feature the DNS server on the EdgeRouter. In the network settings configuration, I just had to change from "automatic" to "manual" and configure the IP of the router. 

### 2.2. DynDNS

I didn't have a fix IP. 

The nice thing about having to keep the provider's box is that it comes with a DynDNS server working out of the box, you don't need to register anywhere else. So I activated the feature and I got following hostname: _mydomain.myproviderbox.country_

## 3. Cloudfare

I already had a Cloudfare account with some DNS records configured for _mydomain.net_. I also owned the domain _mydomain.net_. In cloudfare, I set up an alias by adding a CNAME DNS entry:
```
CNAME	*.mydomain.net	mydomain.myproviderbox.country
``` 

The other thing I needed from Cloudfare was an API token for DNS01 challenge resolution for Let's Encrypt.

Tokens can be created at User Profile > API Tokens > API Tokens. The following settings are recommended:
```
    Permissions:
        Zone - DNS - Edit
        Zone - Zone - Read
    Zone Resources:
        Include - All Zones
```

Created cloudflare.yaml and modified clusterissuer and ingress accordingly. Not working: 
E0515 13:19:00.853223       1 controller.go:140] cert-manager/controller/challenges "msg"="re-queuing item  due to error processing" "error"="error getting cloudflare secret: secret \"cloudflare-api-token-secret\" not found" "key"="default/empanadilla-tls-2412158802-1365669482-910724414" 

(...)

## Links:
https://cert-manager.io/docs/configuration/acme/dns01/cloudflare
