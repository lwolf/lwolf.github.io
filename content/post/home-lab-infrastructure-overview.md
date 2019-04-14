+++
author = "Sergey Nuzhdin"
categories = ["architecture", "docker", "infrastructure", "kubernetes", "k8s"]
date = "2019-01-23"
description = "Overview of the infrastructure in my home lab"
featured = "infra_cover.jpg"
featuredalt = ""
featuredpath = "date"
linktitle = ""
draft = false
images = ["img/2019/01/infra_cover.jpg"]
title = "Home Lab Infrastructure Overview"

+++

# Home Lab Infrastructure Overview
Every software or technology I blog about usually goes through my home lab first. A lot of people usually got surprised when they first hear that I have a multi-node Kubernetes cluster at home. It usually takes some time to tell them about all the machines and networking. Of course, not accounting for the time spent answering the question “why do you need it”.
I added a few new devices and reconfigured everything from scratch recently. I thought that it would be a good time to write up a blog post that I could just send during similar discussions.

## Hardware

A few months ago I bought 3 ODROID-HC1 devices to build a dedicated storage cluster for my k8s. These devices are ARM-based, so I ended up redeploying my entire cluster to make it multi-arch friendly. 
Operating multi-arch cluster change few things about how I build software. For example: now I build everything for at least 3 architectures: ARM64, AMD64 and ARM. Another example - I started building and/or packaging containers that I use in a manifest-based way.  But, this post is not about that.
So, with those ODROIDs I now have the following hardware in my cluster:

| **Device**                                                                                                                              | **RAM** | **CPU**                                                                                | **Storage**           | **Arch** |
| --------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------------------------------------------------------------------------------------- | --------------------- | -------- |
| [Up! Squared](https://up-shop.org/home/182-up-squared-pentium-8gb-128b-pack.html)                                                       | 8gb     | Pentium N4200 4C/4T 2.5 GHz                                                            | 128 emmc              | amd64    |
| [Supermicro Superserver](https://www.supermicro.com/products/system/midtower/5028/sys-5028d-tn4t.cfm) <br>(multiple VMs on VMWare ESXi) | 32gb    | Intel® Xeon D-1541 8C/16T 2Ghz                                                         | 3x6Tb hdd + 2x256 ssd | amd64    |
| **3x** [ODROID-HC1](https://www.hardkernel.com/shop/odroid-hc1-home-cloud-one/)                                                         | 2gb     | Samsung Exynos5422 <br>2.1GHz Quad-Core (Cortex®-A15)<br>1.4GHz Quad-Core (Cortex®-A7) | 256ssd + 32 sdcard    | arm      |

Apart from this, there is a Netgear NAS with 2x3Tb in RAID1 for long term backups which is not part of a k8s cluster.

## Topology

At the moment, the infrastructure looks like the following:

{{< figure src="/img/2019/01/home_infra.png"  alt="home infrastructure plan" >}}

At the edge of the network, there is a Cisco rv325 router. I only got it a few days ago, but the plan is to replace my current PFSense VM with it.

Then there are two wifi routers connected to it:

- BT - router I got from the internet provider. It lives on a separate VLAN and does not have any access to the LAN. Pure internet access for guests.
- Netgear R7000 - primary wifi router connecting every device to the network.

There is another small Netgear switch at the bottom used for a group of ODROID storage nodes.

I'm planning on publishing another post about my experience provisioning multi-arch kubernetes cluster using kubespray. But for now, as usual, let me know if you have any questions.
