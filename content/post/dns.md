+++
math = true
date = "2020-05-19T10:00:00+02:00"
title = "DNS Configuration for K8s"
tags = []
highlight = true

[header]
  caption = ""
+++

__Goal:__

_Configure DNS, so applications running on the K8S cluster are reachable from the internet and TLS-protected_

Setting up a home made kubernetes cluster is quite straightforward. However, for deploying applications or services accessible from the internet, the configuration capabilities of the standard provider's internet boxes are usually too limited. 

In particular I had the issue of the internal host resolution. My internet box wasn't able to properly route requests to the own external IP address from inside the internal network. And not even in the _advanced configuration_ was possible to configure static host mapping. 

I wanted to deploy a web application (cozy) on my cluster. In order to have this working, I needed the following: 

 * A FQDN _example.com_ resolvable from the internet
 * A wildcard SSL certificate covering both _example.com_ and _*.example.com_ 
 * A port forwarding rule for the HTTP and HTTPS ports to the K8S cluster
 * A fix IP or a DynDNS service identifying the host IP where the application is running
 * An own DNS server with a static host or an entry pointing to the internal IP of the FQDN 
 * For certmanager and Let's Encrypt, a DNS provider supporting the DNS01 solver mechanism 

## 1. Ubiquiti EdgeRouter X (SFP) as DNS server

The easiest solution for all this mess would have been to replace the provider's internet box through a router of my own. I had a EdgeRouter X SFP I once bought, but never had time to set up, lying around. However short ago my internet provider replaced the internet box through a new one, featuring 10 GB internet. The EdgeRouter does not support the 10GB internet connection, so to replace the box wasn't an option anymore.

![EdgeX](/techblog/img/edgex.jpg)

So I decided to use the EdgeRouter to work only as a small internal DNS server.

_NOTE: in this post I'm refering to the EdgeRouter as "the router" even if it's not going to be used as one. The actual router for the host network is what I'm refering to as the "internet box"._

### 1.1 Factory reset

I followed the instructions in the quick start guide to do a factory reset of the router: press reset button, connect the power cable and wait until a moment until the lights stop blinking.

### 1.2 Connect to the router

To connect to the router, first I disconnected the wifi on my laptop. 

Then I connected the eth0 port directly to my laptop with a network cable and configured a fix IP on the eth0 port of the laptop as follows:
```
sudo ifconfig eth0 192.168.1.4 netmask 255.255.255.0 up
```

After this I was able to access the router on the browser under 192.168.1.1.

Before proceeding with any configuration, I performed a firmware upgrade on the device. 

### 1.3 Configure switch interface

First of all, I had to configure the _switch_ interface.

The router has following interfaces:

 * Two external interfaces: eth0 (normal ethernet port) and eth5 (the SFP port) 
 * One internal interface _switch_ with bridged ports eth1, eth2, eth3 and eth3 

On the _Dashboard_, for each interface there is a drop-down button labeled _Actions_ located on the right. For _switch0_, after selecting _Config_, I chose to _manually define the IP address_ and set it to  192.168.1.140 / 24. 

Next thing is to deactivate the DHCP server, so the router does not try to assign IP addresses to other devices in the network. We want the router to act just as a small server and not really as a router. The internet box keeps acting as the DHCP server and router for the home network. 

Under _Wizards_, I went to the _Basic Setup_. There, under _LAN ports_, I deactivated the DHCP Server, unchecking the box _Enable the DHCP server_.

On the same wizard, in the _User setup_ section, I changed the default user setting up a new user and password.

After this I pressed _Apply_ to apply the changes and restart the router.

Once the router has restarted, I disconnected the router from my laptop. 

I connected the eth1 port to a switch serving my home network and rebooted (disconnecting the power cord and connecting it again). After this, the router was accessible on the browser under 192.168.1.140. 

### 1.4 DNS configuration

Next step is the effective DNS configuration. The idea was to configure a public name server (as Google) and to manually override static some host mappings to have the right DNS resolution from inside the internal network for my applications on the K8s cluster. 

1. In order for the router to find the default gateway, under _Routing_ I added a new static route with description _internetbox_, destination 0.0.0.0 / 0, next hop 192.168.1.1 (which is the IP of the internet box) and interface _switch0_.


2. In the _Config Tree_ under _system_ there is a _System parameters_ section. There I set up the the _name-server_ to 127.0.0.1. With this the router will now act as the name server.   

3. Under _service_, _dns_ and _forwarding_, I configured _listen-on_ as _switch0_ and as name-servers 8.8.8.8 and 8.8.8.4 (the Google ones).

The EdgeOS was now ready to act as the DNS server for the home network. 

 _NOTE: regarding step 1, another way to do the same is to configure the default gateway in the system configuration._

### 1.4 Add static host mapping
 
The whole point of setting up the own DNS server is to be able to add some static host mappings for IP resolution inside of the network. 

Under _Config Tree_ and _system_ I found the _static-host-mapping_ section. Under _host-name_ I configured the hostname and the aliases I needed for the application:

__Hostname:__ _cozy.example.com_

__IP:__ 192.168.1.131 


__Aliases:__

_home.cozy.example.com_

_drive.cozy.example.com_

_photos.cozy.example.com_

_settings.cozy.example.com_

_store.cozy.example.com_

_mailhog.cozy.example.com_


The IP is the one of the worker's node on the K8s cluster. The hostnames and aliases are the ones needed by the first application I was intending to deploy in K8s:

* [Home-made K8s cluster]({{< ref "/post/kubernetes_cluster" >}})
* [Self-Hosted cozy on K8s]({{< ref "/post/cozy" >}})

## 2. Internet Box

### 2.1. DNS Server Configuration

After I had my DNS server up und running, the remaining thing to do was to change the DNS server configuration on the internet box to feature the DNS server I just set up on the EdgeRouter. In the network settings configuration, I just had to change from _automatic_ to _manual_ and configure the IP of the router, which was 192.168.1.140.

I also added a static lease so the router would always get the same IP address. 

### 2.2. DynDNS

I don't have a fix IP. I still needed a FQDN.

The bright side of having to keep the provider's box is that it comes with a DynDNS server working out of the box, so you don't need to register a dedicated account for this. So I activated the feature and I got following hostname: _example.myproviderbox.country_

### 2.3. Port Forwarding and Firewall

To make the application accessible from the internet, I had to:

 * Modify the default firewall configuration to allow inbound traffic to port 443
 * Create a port forwarding rule to forward all requests to port 443 to the internal IP address 192.168.1.131 (which was the address of the worker node where nginx was running).

## 3. Cloudfare

I have a Cloudfare account and a registered domain. In cloudfare, I set up an alias by adding a CNAME DNS entry, pointing to the internet name I got from the DynDNS:
```
CNAME	example.com	example.myproviderbox.country
CNAME	*.example.com	example.com
``` 

The other thing I needed from Cloudfare was an API token for DNS01 challenge resolution for Let's Encrypt.

Tokens can be created under _User Profile_ > _API Tokens_ > _API Tokens_. I used the recommended settings:
```
    Permissions:
        Zone - DNS - Edit
        Zone - Zone - Read
    Zone Resources:
        Include - All Zones
```

With these preparations, my home network was ready to host and route requests to applications deployed on the K8s cluster.

## Links:
[EdgeRouter X](https://dl.ubnt.com/guides/edgemax/EdgeRouter_ER-X-SFP_QSG.pdf)

[Cloudflare](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare)

