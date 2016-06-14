+++
author = "Sergey Nuzhdin"
categories = ["infrastructure", "docker", "infrastructure"]
date = "2016-06-25"
description = ""
featured = "kubernetes.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
draft = true
title = "Migrate infrastructure to Kubernetes. Basic services."

+++

In previous [post](/post/migrate-infrastructure-to-kubernetes-building-baremetal-cluster/)
I finished description of installation of kubernetes cluster on bare-metal hardware.
At this point we should be able to communicate with it using `kubectl`

In this one I will go through installation of basic services to use and monitor cluster.
For example DNS, heapster and different dashboards.
<!-- more -->


# Deploying addon services

Kubernetes comes with several very useful addons, available in its github,
either in [kubernetes](https://github.com/kubernetes/kubernetes) or in [contrib](https://github.com/kubernetes/contrib).
But all this addons still needs to be installed.

Before we begin with this, we need to do one preparation step - create system namespace.
By default, Kubernetes has only one [namespace](http://kubernetes.io/docs/user-guide/namespaces/), called `default`.
But most of addons avaliable in the internet requires you to have separate namespace for system needs.
It should be called `kube-system`.

## Creating system namespace
As far as I heard Kubernetes is going to create it automatically from version 1.3.
So, first, lets check namespaces we have in our cluster.
Getting namespaces is as simple as getting any other type of objects inside Kubernetes:

```bash
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    53d
```

If `kube-system` is not among them we need to create it.
To do it we need to create simple yaml(or json) file and send it to Kubernetes.
Lets create file `kube-system.yaml` with the following content:

```yaml
# kube-system.yaml

kind: Namespace
apiVersion: v1
metadata:
  name: kube-system

```

Its simple and self-explanatory: create new entity of type Namespace with the name kube-system.
Now we need to deploy it to the cluster:

```
kubectl create -f kube-system.yaml
```

Thats it, after this running `kubectl get namespaces` should return two namespaces.


## Deploying DNS service

This one is also part of the standard addons available in [Kubernetes repository](https://github.com/kubernetes/kubernetes).
To deploy it we need to ip address of DNS and the name of the cluster we used in during minions creating.
If we're using default ip address for DNS server (10.100.0.10) and
default cluster name (cluster.local) we can just deploy it as is.
Otherwise we need to change `skydns-rc.yaml` to match our settings.

```
# kubernetes/cluster/addons

kubectl --namespace=kube-system create -f skydns-rc.yaml
kubectl --namespace=kube-system create -f skydns-svc.yaml
```

### Deploy Kubernetes dashboards

Now we want to have nice dashboards to monitor our cluster state and may be even deploy new services.
For this, Kubernetes has two default dashboards. First one is the part of addons,
and the other one is has its own [repository](https://github.com/kubernetes/dashboard).

Lets deploy both dashboards:

```bash
# kubernetes/cluster/addons
kubectl create -f kube-ui/

kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

To access it we need to run `kubectl cluster-info` to find out urls.

```bash
> $ kubectl cluster-info
Kubernetes master is running at https://10.10.30.11:443
KubeDNS is running at https://10.10.30.11:443/api/v1/proxy/namespaces/kube-system/services/kube-dns
KubeUI is running at https://10.10.30.11:443/api/v1/proxy/namespaces/kube-system/services/kube-ui
kubernetes-dashboard is running at https://10.10.30.11:443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```

### Deploying heapster

[Heapster](https://github.com/kubernetes/heapster) is one of the systems used in Kubernetes to collect metrics.

```bash
git clone https://github.com/kubernetes/heapster
cd heapster
kubectl create -f deploy/heapster
```

Heapster uses grafana dashboard to show its metrics.
But it also has another dashboard - kubedash, which is pretty simple and nice for quick overview of resource usage.

```
git clone https://github.com/kubernetes/kubedash
cd kubedash
kubectl create -f deploy/bundle.yaml
```

After deploy, kubedash will be available on `https://<kubernetes-master>/api/v1/proxy/namespaces/kube-system/services/kubedash/`.

{{< figure src="/img/2016/06/kubedash.png" alt="Kubedash dashboard" >}}


### Deploy newrelic daemon
I'm using newrelic service all the time to monitor my applications and servers.
So I'm going to run it as a daemon on all minions.

Newrelic configuration I'm going to deploy, with good readme btw, could be found in [kubernetes/examples](#).

Shortly, to run newrelic you need to create config file and then create a base64 hash of it, after this you can deploy it.
We need linux machine to create base64 hash of our config and then deploy it.
```
kubectl create -f newrelic-config.yaml
kubectl create -f newrelic-daemonset.yaml --validate=false
```


### Deploy fluentd-logging
```
kubectl create -f fluentd-elasticsearch/
```

