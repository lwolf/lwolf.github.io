+++
author = "Sergey Nuzhdin"
categories = ["architecture", "docker", "infrastructure"]
date = "2016-04-25"
description = ""
featured = "kubernetes.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
draft = true
title = "Move infrastructure to Kubernetes. Basic services."

+++

In previous [post](#) I finished description of installation and configuation of kubernetes cluster.
We also can talk to our cluster using `kubectl`.
In this one I'm going to talk about installation on basic services into it.
For example DNS, heapster and different dashboards.
<!-- more -->


# Deploying addon services

### Creating system namespace

Kubernetes comes with several very useful addons, which still needs to be installed.
Kubernetes itself has only one [namespace](#) called `default` but for most of the addons
and other system services it requires another one called `kube-system`.

You can check which namespaces you already have in your cluster bu running `kubectl get namespaces`.

If kube-system is not among them you need to create it to be able to deploy addons.
To do it you need to create simple yaml(or json) file and run it.

Lets say you have file `kube-system.yaml` with the following content.

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: kube-system

```

Now you need to deploy it to the cluster:

```
kubectl create -f kube-system.yaml
```

Thats it, after this if you run `kubectl get namespaces` you will have one more namespace.

### Deploying DNS service
This one is also part of the standard addons available in kubernetes repository.
If you're using default ip address for dns (10.100.0.10) and
default cluster name (cluster.local) you can just deploy it from there.
Otherwise you need to change rc file to match your settings.

```
cd kubernetes/cluster/addons

kubectl --namespace=kube-system create -f skydns-rc.yaml
kubectl --namespace=kube-system create -f skydns-svc.yaml
```

### Deploy kubernetes dashboards

Now we want to have nice dashboards to monitor our cluster and may be even deploy new services.
For this kubernetes has two default dashboards. One is the part of addons,
and the other one is like new one, which has its own [repository](#).

```
cd kubernetes/cluster/addons
kubectl create -f kube-ui/

kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

To access first one you need to run `kubectl cluster-info` to find out its url.
The second one will be deployed to random node and port,
and to find it out you can run `kubectl describe service kubernetes-dashboard --namespace=kube-system`

### Deploying heapster

[Heapster](#) is one of the systems used in kubernetes to collect metrics.

```
git clone heapster
cd heapster
kubectl create -f deploy/heapster
```
Heapster uses grafana dashboard to show its metrics, but there is also another dashboard available - kubedash

```
git clone kubedash
cd kubedash
kubectl create -f deploy/bundle.yaml
```

To find out where kubedash is available, run - `kubectl describe ????`


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

