+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "kubernetes", "k8s", "devops"]
date = "2019-02-02"
description = ""
featured = "kubespray-logo.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
images = ["img/2019/02/kubespray-logo.png"]
title = "How to deploy multi-arch Kubernetes cluster using Kubespray"
+++


# How to deploy multi-arch Kubernetes cluster using Kubespray
I recently bought 3 [ODROID-HC1](https://www.hardkernel.com/shop/odroid-hc1-home-cloud-one/) devices to add a dedicated storage cluster to my home Kubernetes. I thought that it’s a good excuse to spend some time redeploying the cluster. Usually, I would’ve gone with CoreOS, since I’m a big fan of their immutable OS. Unfortunately, that is not an option if you have ARM nodes. So I had to choose between manual provisioning and Ansible. I chose Ansible. 

I knew about [Kubespray](https://github.com/kubernetes-incubator/kubespray), but still, I decided to spend some time looking for alternatives. I found a few other projects: some targeted towards ARMs, some with a “simplified” approach. In the end, I decided to go with Kubespray. Partly, because it is a part of official k8s tooling, partly because they started to use kubeadm under the hood.

Kubespray is the oldest project aimed to automate Kubernetes cluster provisioning. There are a lot of good manuals on how to deploy a simple k8s cluster using Kubespray, like [this](https://dzone.com/articles/kubespray-10-simple-steps-for-installing-a-product) or [this](https://medium.com/@iamalokpatra/deploy-a-kubernetes-cluster-using-kubespray-9b1287c740ab). 
So, I’m not going to repeat any of this here. Instead, I'll try to highlight the steps required to make things work in a multi-arch cluster.


## Multi-arch deployment

Kubernetes’s support for non-amd64 platforms improved dramatically during the last 2 years. Thanks to the popularity of cheap ARM boards like RaspberryPi and the community. Kubespray does not yet work with multi-arch setup out of the box, but it's getting there with every release.
As it turned out it’s not very difficult to make it work.
In general, the work could be split into a few categories:


- change some Kubespray configs where architecture is still hardcoded.
- choose overlay network that works with different architectures and setup it
- configures arch-specific software to be scheduled to corresponding nodes using **nodeSelector**


## Fine-tuning settings

When I started experimenting with Kubespray and multi-arch deployment, around 2 months ago, I had to change a lot of files. Lots of URLs for binaries and docker images had a hardcoded amd64 part. Luckily, at the moment of writing only a few such places left in the repo.


1. One of such hardcoded arch is still present in the latest 2.8 release, but it's already fixed in master. So we need to change it.

```
    - hyperkube_download_url: "https://storage.googleapis.com/.../linux/amd64/hyperkube"
    + hyperkube_download_url: "https://storage.googleapis.com/.../linux/{{ image_arch }}/hyperkube"
    - etcd_download_url: "https://github.com/coreos/.../etcd-{{ etcd_version }}-linux-amd64.tar.gz"
    + etcd_download_url: "https://github.com/coreos/.../etcd-{{ etcd_version }}-linux-{{ image_arch }}.tar.gz"
```

Previously I had to do such changes all over the place.


2. Another thing that you have to deal with is the current implementation of checksums map. Kubespray was not meant to run in a hybrid environment, so it assumes that a binary could only have a single checksum. That is not a case in a multi-arch environment, where you have one binary per architecture.
3. It might change in the future. For example, a simple map could be replaced with some sort of multileveled map. But for now, the easiest way is to comment out sha256 checksums for the binaries you’re gonna use.

```
    # roles/download/defaults/main.yaml
      kubeadm:
        enabled: "{{ kubeadm_enabled }}"
        file: true
        version: "{{ kubeadm_version }}"
        dest: "{{local_release_dir}}/kubeadm"
    #    sha256: "{{ kubeadm_binary_checksum }}"
        url: "{{ kubeadm_download_url }}"
        unarchive: false
        owner: "root"
        mode: "0755"
        groups:
          - k8s-cluster
```


3. The last thing left in this section is to extend the architecture group with ARM variable.

```
    # roles/kubernetes/preinstall/tasks/0040-set_facts.yml
    - set_fact:
        architecture_groups:
          x86_64: amd64
          aarch64: arm64
          armv7l: arm
```


## Overlay network

The main choice one has to make is what overlay network to use. 
Here are the main options I considered:

* ***cilium*** was my first choice. I heard a lot of great stuff about it and for a long time, I want to try it on a real system. Unfortunately, it [does not support ARM yet](https://github.com/cilium/cilium/issues/1104).
* **calico** -  is another popular solution which I didn’t try yet. But, again, as far as I can tell, it still does not support multi-arch containers, but they are [working on it](https://github.com/projectcalico/calico/issues/1865). [**UPDATE**: the ticket is closed now, and you can use it on amd64/arm64 and ppc64 architectures.]
* ***flannel***  - I’ve been using flannel in most of my k8s clusters for a few years now and it turned out that I’m gonna use it for a much longer period. It is the only CNI that supports all the architectures. Well, maybe not all, but all architectures that I would ever need.

## Flannel

Deploying flannel is very straight-forward. Flannel does not yet support multi-arch containers. You need to add a DaemonSet for each architecture you use and limit the nodes using NodeSelector. For example, to make it work on ARM64, all you need to do is copy original manifest and update a few lines.

```
    apiVersion: extensions/v1beta1
    kind: DaemonSet
    metadata:
      name: kube-flannel-ds-arm64
      namespace: kube-system
      ...
    spec:
      template:
        spec:
          ...
          nodeSelector:
            beta.kubernetes.io/os: linux
            beta.kubernetes.io/arch: arm64
          ...
          containers:
          - name: kube-flannel
            image: {{ flannel_image_repo }}:{{ flannel_image_tag }}-arm64
```          

When I added all the changes for the flannel and tried to provision, playbook kept failing with the following error.

```
    TASK [kubernetes/secrets : Gen_certs | add CA to trusted CA dir] ********************************************************************************************************************
    Saturday 29 September 2018  11:23:52 +0100 (0:00:00.536)       0:09:27.463 ****
    changed: [node1] => {"changed": true, "checksum": "6133a4cde211b2699082947ed5877627fc17a5fb", "dest": "/usr/local/share/ca-certificates/kube-ca.crt", "gid": 0, "group": "root", "md5sum": "538b74ed3d841c835d026dd51de99882", "mode": "0644", "owner": "root", "size": 1094, "src": "/etc/kubernetes/ssl/ca.pem", "state": "file", "uid": 0}
    changed: [node2] => {"changed": true, "checksum": "6133a4cde211b2699082947ed5877627fc17a5fb", "dest": "/usr/local/share/ca-certificates/kube-ca.crt", "gid": 0, "group": "root", "md5sum": "538b74ed3d841c835d026dd51de99882", "mode": "0644", "owner": "root", "size": 1094, "src": "/etc/kubernetes/ssl/ca.pem", "state": "file", "uid": 0}
    fatal: [odroid1]: FAILED! => {"changed": false, "msg": "Source /etc/kubernetes/ssl/ca.pem not found"}
```

Looking into failing step, it does not seem to be related to the networking. But in the logs of the several scheduled containers, it’s clear that the problem is in CNI plugins.

```
    Normal   Scheduled               48m                 default-scheduler  Successfully assigned cert-manager/cert-manager-695f7b5bdc-t28j7 to odroid1
    Warning  FailedCreatePodSandBox  48m                 kubelet, odroid1   Failed create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "9c887728f495874b0b9efd444900f829283b26884bf23d4255193bdbd8139f4d" network for pod "cert-manager-695f7b5bdc-t28j7": NetworkPlugin cni failed to set up pod "cert-manager-695f7b5bdc-t28j7_cert-manager" network: failed to find plugin "loopback" in path [/opt/cni/bin], failed to clean up sandbox container "9c887728f495874b0b9efd444900f829283b26884bf23d4255193bdbd8139f4d" network for pod "cert-manager-695f7b5bdc-t28j7": NetworkPlugin cni failed to teardown pod "cert-manager-695f7b5bdc-t28j7_cert-manager" network: failed to find plugin "portmap" in path [/opt/cni/bin]]
    Normal   SandboxChanged          2m (x204 over 48m)  kubelet, odroid1   Pod sandbox changed, it will be killed and re-created.
```

After some googling, I found a few issues in the flannel repo mentioning missing plugins in the flannel 0.10. Especially the absence of the [portmap plugin](https://github.com/coreos/flannel/issues/890#issuecomment-349448629). The workaround for this is either downgrade to flannel 0.9.1 or install [CNI plugins](https://github.com/containernetworking/plugins) manually. I decided to install the plugins using Ansible. After this, provisioning succeeded. 


## Flannel update

When you’re dealing with a multi-arch setup on daily basis at some point you'll get tired of gating deployments. I mean for each deployment you create, you need to add NodeSelector so it will be able to run.

Docker's manifests are still not very popular, and only a few projects are using. As far as I know, only Docker and GKE registries support them.

So I created a [repo on GitHub](https://github.com/lwolf/docker-multiarch) with a bunch of dockerfiles and build steps to build manifest-based containers for the software I use. The build is automatic, it checks releases daily and runs build on new releases. 
Currently, I have flannel, flannel-cni, kubernetes-dashboard, Prometheus and helm. 
Thanks to the manifest-based flannel container, I removed my custom one-per-architecture flannel DaemonSet. Now I use an upstream version of it. So, instead of a bunch of DaemonSets, I need to replace a few CNI related variables:

```
    # roles/download/defaults/main.yaml
    flannel_cni_version: "v0.7.4"
    flannel_image_repo: "lwolf/flannel"
    flannel_cni_image_repo: "lwolf/flannel-cni"
```

## Upgrading K8S versions

During the time have this setup, a few versions of Kubernetes were released. So I had a good chance to test upgrades between releases. I started with v1.12 and went through the upgrades up to v1.12.5. In general, everything went well, I didn’t have any problems with the core components. The only thing that was causing provision to fail from time to time was [nginx-ingress-controller](https://github.com/kubernetes/ingress-nginx) provided by Kubespray. For some reason, Kubespray is trying to delete it and reinstall each time. I ended up disabling it and other charts.  I wasn’t planning on managing helm charts with Kubespray anyways. I find this workflow a bit strange: Ansible variables -> jinja -> helm templates -> helm install.

After this change, updates became stable.


## Conclusions

It’s great to see that Kubernetes is getting good support for a multi-arch setup. I am very pleased with the number of hacks that I had to do to Kubespray to make it work seamlessly in a hybrid cluster. 

I planned to cover the deployment of GlusterFS in this post as well since it was the reason for this whole deployment. 

But GlusterFS, as usual, caused a lot of trouble to deploy and deserves a separate post.
