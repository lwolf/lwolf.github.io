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
title = "Migrate infrastructure to K8s. Part 3. Load Balancing."

+++


<!-- more -->


# Configuring ingress and loadbalancing
```
kubectl create -f ./ingress-lb/default-backend.yaml
kubectl expose rc default-http-backend --port=80 --target-port=8080 --name=default-http-backend
kubectl create -f ./ingress-lb/nginx-ingress-daemonset.yaml
```
