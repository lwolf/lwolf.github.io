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
title = "Move infrastructure to Kubernetes. LoadBalancing."

+++


<!-- more -->


# Configuring ingress and loadbalancing
```
kubectl create -f ./ingress-lb/default-backend.yaml
kubectl expose rc default-http-backend --port=80 --target-port=8080 --name=default-http-backend
kubectl create -f ./ingress-lb/nginx-ingress-daemonset.yaml
```


# Testing/Staging environment

## install needed services
  * create separate namespace
  * install gitlab
  * install jenkins
  * install docker-hub

## create repositories and build jobs
## Configure autobuild/autodeploy to staging
