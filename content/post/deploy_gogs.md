+++
math = true
date = "2020-03-22T15:00:00+02:00"
title = "First Kubernetes Deployment"
tags = []
highlight = true

[header]
  caption = ""
  image = ""

+++

__Goal:__

_Deploy Gogs - a painless self-hosted Git service - on homemade Kubernetes Cluster._

After I had my own Kubernetes cluster ready, I wanted to try it out. Setting up the cluster was almost too easy, so I was expecting some fine-tuning to be necessary. As first application I chose gogs - a self-hosted github clone. 

I struggle with too many things to do in very little time - like many others, I guess - so I'm always trying to optimise life processes. The idea at this point was to use the gogs Kanban board to adopt an agile approach for private TODOs and tasks. 

Tasks:

1. Create Storage Class
2. Deploy Gogs 

## 1. Set up Storage Class

First of all, make sure the client machine can reach the cluster via kubectl:
```
eramon@caipirinha:~/dev/kubernetes$ kubectl cluster-info
Kubernetes master is running at https://192.168.1.126:6443
KubeDNS is running at https://192.168.1.126:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

The prerequisites for installing Gogs said:
_PV support on underlying infrastructure (if persistence is required)_

In general, in order for our future pods to be able to claim persistent volumes for their applications, we need to set up a storage class for the cluster.

Since this is a test setup, I just wanted a StorageClass using the local disk for storage of the persistent volumes:
```
eramon@caipirinha:~$ vi storageClass.yaml
```

This is the content of the file:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Apply:
```
eramon@pacharan:~$ kubectl apply -f storageClass.yaml
storageclass.storage.k8s.io/local-storage created
```

## 2. Deploy gogs 
_TODO It didn't work using Helm charts. Try to deploy directly using yaml files._


## Links:
[Installing Helm](https://helm.sh/docs/intro/install/)
[Helm Releases @Github](https://github.com/helm/helm/releases)
[Helm Quickstart Guide](https://helm.sh/docs/intro/quickstart)
[Gogs Helm Chart @Github](https://github.com/helm/charts/tree/master/incubator/gogs)

