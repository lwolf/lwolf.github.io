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
title = "Move infrastructure to Kubernetes"

+++

I started trying to switch to docker about a year ago, but all tools were kind of `not-production-ready`.
With docker-compose it was unreal to scale containers without restart. Deis looked like a black box with a lot of magic.
Then tutum appeared and it was awesome, it was, really, the first working solution. So I switched to it.
It was fine most of the time, yes it had problems with networking, yes it was annoying to copy-paste
all environment variables into each container, but it was in beta, and it was free.
But then docker bought it and decided to charge $15 per node. Yes, it became more stable, with better networking.
But for me it wasn't doing much for me since I'm running everything on baremetal servers.

And since I was needed to migrate from tutum and redo my infrastructure anyways either to docker-cloud
or to something else. Among options there were, except for docker-cloud, raw docker's compose/swarm and kubernetes.
So, I decided to go away from docker-cloud.

<!-- more -->

# First steps with Kubernetes

First time I tried to use kubernetes even before tutum. I think v1.0 was just released when I tried it.
After few weeks playing with it, I decided to not use it in production yet. Mostly because of difficulties
with loadbalancing on baremetal.
Few weeks ago I was responsible for choosing docker management platform at work and for building cluster on it.
So I spent some time investigating current solutions and ended up building kuberenetes cluster on CoreOS.

After successfull built of test multinode cluster and dev-environment at work
I spent weekend building cluster for my own projects.
This will not be the step-by-step tutorial but more like sumarization of my experience.

Basicly for my side projects, I have one physical server with ESXi and a lot of virtual machines.
Before tutum I used to create one VM per project or service, and ended up with about 20 VMs.
With tutum I reduced this number to about 8-10. Only 4 of them were related to tutum: 3 VMs for tutum, 1 vm as router.
Other VMs had some lagacy parts which I didn't want to move to docker for some reasons.

# Building cluster
With this switch from tutum to kubernetes I wanted to do it as `production-like` as possible.
Since I'm going to use CoreOS I want to use its awesome autoupdate feature, and for this I need to have
several nodes of each type: etcd, kubernetes-masters, kubernetes-minions. Since it will be small cluster
and I do not want to waste resources, I'm going to have 6 VMs for kubernetes and 1 for external loadbalancer.
3 of it will run etcd + kubernetes masters and 3 will be minions. Having 3 minions could sound like a waste of resources,
but, remember, I want to have production-like solution and with 1 minion it will be to easy to go with some stupid solutions,
like I have only 1 machine, I can use local hard drive for storage or hardcode IP address and port of this node in loadbalancer.
Also I want to be able to easily scale it to several machines.

Here is a schema of what I'm going to build.
{{< figure src="/img/2016/04/kube-cluster-schema.png" alt="Kubernetes cluster schema" >}}

On schema I have two loadbalancer, but in fact its one.

To achive this I need to have

* virtual/physical machines
* dhcp + (i)pxe for initial boot and install of coreos
* nginx or haproxy for loadbalancing and as ipxe config provider
* write cloud-configs to configure coreos



## Configuring dhcp and iPXE
First step we need to do, after creating VMs, is configure DHCP and iPXE to be able to boot and install CoreOS.
There are a lot of manuals about how to configure dhcp in ubuntu (I'm running ubuntu on that machine) I'm not going to copy-paste it.
One thing I want to mention is that I hardcoded MAC/IP addresses of 3 my machines which I'm going to use as ETCd/Kubernetes masters.

here is example of dhcp record cofigured for ETCD master:

```
  host kube-etcd-01 {
      hardware ethernet 01:23:45:67:89:00;
      fixed-address 10.10.30.11;

      if exists user-class and option user-class = "iPXE" {
          filename "http://10.10.30.1:8000/ipxe/ipxe-kube-etcd-01";
      } else {
          filename "undionly.kpxe";
      }
   }
```

and here is example of the `ipxe-kube-etcd-01` config:

```
#!ipxe

set base-url http://10.10.30.1:8000
kernel ${base-url}/images/coreos_production_pxe.vmlinuz cloud-config-url=${base-url}/ipxe/cloud-config-bootstrap-etcd-01.sh sshkey="<YOUR_SSH_KEY>"
initrd ${base-url}/images/coreos_production_pxe_image.cpio.gz
boot

```

At this file we can see that we will run coreos image with cloud-config and ssh key.
Having your `sshkey` here is useful, when for some reason provisioning/install was not successfull and you need to debug.
Also as you can see we're telling coreos to use `sh` script as cloud-config. This is done because I want to run coreos-install after boot.
Here is content of `cloud-config-bootstrap-etcd-01.sh`

```
#!/bin/bash

curl http://10.10.30.1:8000/ipxe/cloud-config-etcd-01.yml -o cloud-config.yaml
sudo coreos-install -d /dev/sda -c cloud-config.yaml -C alpha
sudo reboot
```

This script just downloads real cloud config, installs coreos on disk and reboot.
This also leads to maintaining 3 files for each type of machine, which I really don't like and plan to change/automate.

## Configuring nginx to serve cloud configs
IPXE can serve configs from many locations, I prefer to use http. And since its just plain text files having nxinx is enough for this.
Most of example cloud-cofigs you can find in the internet has references to `$private_ipv4` or `$public_ipv4` variables.
If you're using cloud providers for your coreos setup, this variables will be translated to real addresses. But since we using baremetal setup we
need to impletement something to have similar behaviour. The easiest way is to use nginx for this.

Having `sub_filter $public_ipv4 '$remote_addr';` rule in your nginx location block will put ip address of the host requesting the config instead of variable on the fly.

And again coreos documentaion has example of nginx [config location block](https://coreos.com/os/docs/latest/nginx-host-cloud-config.html)


## Generate ssl certificates for master node and admin user
Generation of ssl certificates for kubernetes is described in details on [coreOS site.](https://coreos.com/kubernetes/docs/latest/openssl.html).
So I will just show my ssl config and commands that needs to be run.

openssl.cnf

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = kubernetes.example.com <- my public domain name
IP.1 = 10.100.0.1   <- cluster master ip
IP.2 = 10.10.30.11  <- master/etcd 01
IP.3 = 10.10.30.12  <- master/etcd 02
IP.4 = 10.10.30.13  <- master/etcd 03
IP.5 = 10.10.30.1 <- loadbalancer IP
IP.6 = 111.111.111.111 <- my public IP

```

```
$ openssl genrsa -out ca-key.pem 2048
$ openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"
$ openssl genrsa -out apiserver-key.pem 2048
$ openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf
$ openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf

# generate admin key to be able to control cluster

$ openssl genrsa -out admin-key.pem 2048
$ openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=kube-admin"
$ openssl x509 -req -in admin.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out admin.pem -days 365
```


## Writing cloud-configs

At this point we have everything in place, except for cloud-configs.
I'm not going to describe all parts of cloudconfigs, [coreos](https://coreos.com/os/docs/latest/cloud-config.html) did it really great

Lets start from etcd/master node. There are two ways to configure etcd cluster.
 First is to use initial seed cluster - basicaly you know the nodes and IP addresses of your etcd cluster and just hardcode it in config.
 The second one is to use coreos cloud discovery.

In depth comparison is available [here](https://coreos.com/etcd/docs/latest/clustering.html)

Since I know IP addresses of all my master nodes I'm going to use initial seed cluster.

My etcd configuration block looks like this:

```yaml
#cloud-config

hostname: etcd01
coreos:
  etcd2:
    name: etcd01
    listen-client-urls: http://$public_ipv4:2379,http://127.0.0.1:2379
    advertise-client-urls: http://$public_ipv4:2379
    initial-cluster-token: my-kubernetes-cluster
    listen-peer-urls: http://$public_ipv4:2380
    initial-advertise-peer-urls: http://$public_ipv4:2380
    initial-cluster: etcd01=http://10.10.30.11:2380,etcd02=http://10.10.30.12:2380,etcd03=http://10.10.30.13:2380
    initial-cluster-state: new
```

Now lets take a look what we need to add here to run kubernetes master services on the same node:

* We need to put our ssl certificates
* Download kubernetes binaries
* Write systemD units


Lets start with certificates.
To put it on the machine we can either download it from somewhere during boot or just hardcode it into cloudconfig.
For now I'm going to use second option.

To do this, lets add `write-files:` block to our cloudconfig:
Since we already generated all ssl certificates, we just need to copy-paste the content to corresponding blocks.

```yaml
  - path: /etc/kubernetes/ssl/ca.pem
    owner: core
    permissions: 0644
    content: |
      -----BEGIN CERTIFICATE-----
      -----END CERTIFICATE-----

  - path: /etc/kubernetes/ssl/apiserver-key.pem
    owner: core
    permissions: 0644
    content: |
      -----BEGIN CERTIFICATE-----
      -----END CERTIFICATE-----

  - path: /etc/kubernetes/ssl/apiserver.pem
    owner: core
    permissions: 0644
    content: |
      -----BEGIN CERTIFICATE-----
      -----END CERTIFICATE-----

```

Having something in `write-files` is a convenient way for adding something to the system during boot.

Lets create write another file, which will be responsible for checking availability of needed port.

```yaml
  - path: /opt/bin/wupiao
    permissions: '0755'
    content: |
      #!/bin/bash
      # [w]ait [u]ntil [p]ort [i]s [a]ctually [o]pen
      [ -n "$1" ] && \
        until curl -o /dev/null -sIf http://${1}; do \
          sleep 1 && echo .;
        done;
      exit $?
```

Next we need to download kubernetes binaries, lets write another file for this

```yaml
  - path: /opt/bin/kubernetes-install.sh
    owner: root
    permissions: 0755
    content: |
      #! /usr/bin/bash
      sudo mkdir -p /etc/kubernetes/manifests/
      sudo mkdir -p /etc/kubernetes/ssl/
      sudo mkdir -p /etc/kubernetes/addons/

      if [ ! -f /opt/bin/kubelet ]; then
        echo "Kubenetes not installed - installing."

        export K8S_VERSION=$(curl -sS https://storage.googleapis.com/kubernetes-release/release/stable.txt)

        # Extract the Kubernetes binaries.
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubectl
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubelet
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-apiserver
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-controller-manager
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-scheduler
        sudo chmod +x /opt/bin/kubelet /opt/bin/kubectl /opt/bin/kube-apiserver /opt/bin/kube-controller-manager /opt/bin/kube-scheduler
      fi
```

Script is simple, create needed folders and if download binaries if needed.
K8S_VERSION will have the latest stable version of kubernetes. We will configure coreos to run this script during boot.
Now when all files are in place we just need to configure systemD units to start everything.

```yaml
coreos:
  ...
  units:
    - name: kubernetes-install.service
      runtime: true
      command: start
      content: |
        [Unit]
        Description=Installs Kubernetes tools

        [Service]
        ExecStart=/opt/bin/kubernetes-install.sh
        RemainAfterExit=yes
        Type=oneshot
    - name: setup-network-environment.service
      command: start
      content: |
        [Unit]
        Description=Setup Network Environment
        Documentation=https://github.com/kelseyhightower/setup-network-environment
        Requires=network-online.target
        After=network-online.target

        [Service]
        ExecStartPre=-/usr/bin/mkdir -p /opt/bin
        ExecStartPre=/usr/bin/curl -L -o /opt/bin/setup-network-environment https://github.com/kelseyhightower/setup-network-environment/releases/download/1.0.1/setup-network-environment
        ExecStartPre=/usr/bin/chmod +x /opt/bin/setup-network-environment
        ExecStart=/opt/bin/setup-network-environment
        RemainAfterExit=yes
        Type=oneshot
```
Lets start with this two. First one will run our kubernetes installer, and the second one will create network environment file.
Which is very helpful in some cases. For example you can have your current IP address as environment variable.

And of course we need to run kubernetes services:

```yaml
    - name: kube-apiserver.service
      command: start
      content: |
        [Unit]
        Description=Kubernetes API Server
        Documentation=https://github.com/kubernetes/kubernetes
        Requires=setup-network-environment.service etcd2.service fleet.service docker.service flanneld.service kubernetes-install.service
        After=setup-network-environment.service etcd2.service fleet.service docker.service flanneld.service kubernetes-install.service

        [Service]
        EnvironmentFile=/etc/network-environment
        ExecStartPre=-/usr/bin/mkdir -p /opt/bin
        ExecStartPre=/opt/bin/wupiao 127.0.0.1:2379/v2/machines
        ExecStart=/opt/bin/kube-apiserver \
          --admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota \
          --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem \
          --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem \
          --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem \
          --client-ca-file=/etc/kubernetes/ssl/ca.pem \
          --apiserver-count=3 \
          --advertise-address=${DEFAULT_IPV4} \
          --allow_privileged=true \
          --insecure_bind_address=0.0.0.0 \
          --insecure_port=8080 \
          --kubelet_https=true \
          --secure_port=443 \
          --service-cluster-ip-range=10.100.0.0/16 \
          --etcd_servers=http://127.0.0.1:2379 \
          --bind-address=0.0.0.0 \
          --cloud_provider="" \
          --logtostderr=true \
          --runtime_config=api/v1
        Restart=always
        RestartSec=10
    - name: kube-controller-manager.service
      command: start
      content: |
        [Unit]
        Description=Kubernetes Controller Manager
        Documentation=https://github.com/kubernetes/kubernetes
        Requires=kubernetes-install.service kube-apiserver.service
        After=kubernetes-install.service kube-apiserver.service
        Requires=setup-network-environment.service
        After=setup-network-environment.service

        [Service]
        EnvironmentFile=/etc/network-environment
        ExecStartPre=/opt/bin/wupiao ${DEFAULT_IPV4}:8080
        ExecStart=/opt/bin/kube-controller-manager \
          --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem \
          --root-ca-file=/etc/kubernetes/ssl/ca.pem \
          --master=${DEFAULT_IPV4}:8080 \
          --cloud_provider="" \
          --pod_eviction_timeout=30s \
          --leader-elect \
          --logtostderr=true
        Restart=always
        RestartSec=10
    - name: kube-scheduler.service
      command: start
      content: |
        [Unit]
        Description=Kubernetes Scheduler
        Documentation=https://github.com/kubernetes/kubernetes
        Requires=kubernetes-install.service kube-apiserver.service
        After=kubernetes-install.service kube-apiserver.service
        Requires=setup-network-environment.service
        After=setup-network-environment.service

        [Service]
        EnvironmentFile=/etc/network-environment
        ExecStartPre=/opt/bin/wupiao ${DEFAULT_IPV4}:8080
        ExecStart=/opt/bin/kube-scheduler --leader-elect --master=${DEFAULT_IPV4}:8080
        Restart=always
        RestartSec=10

```

Everything here is relatively simple, and could be found in almost eny example of cloud config. But few things I want to highlight.
`${DEFAULT_IPV4}` variable is populated by `setup-network-environment.service` we created earlier. We can use either this or our nginx configuration.
Also I have `--insecure_bind_address=0.0.0.0` inside `kube-apiserver.service`,
because currently I have isolated network, and my nodes can't be accessed from the internet. Otherwise it should contain your internal IP or localhost.

The last thing, is `--apiserver-count=3`. If you have more than one master, you should set this parameter to avoid problems.

Complete cloud-config could be found in [github repository](#)

That's all for masters. At this stage we can start all 3 master master-nodes.
After several minutes you should have working etcd cluster with kubernetes cluster.
But we have not minions, only masters. So lets go through configuratino of minions.


#### Configuring minions
Since we want to have any number of minions and be able to add/remove it at any time, config should be the same for all, without any hardcoding.
That's why will generate client's ssl certicates during install.

```yaml
  - path: /etc/kubernetes/ssl/worker-openssl.cnf
    owner: root
    permissions: 0755
    content: |
      [req]
      req_extensions = v3_req
      distinguished_name = req_distinguished_name
      [req_distinguished_name]
      [ v3_req ]
      basicConstraints = CA:FALSE
      keyUsage = nonRepudiation, digitalSignature, keyEncipherment
      subjectAltName = @alt_names
      [alt_names]
      IP.1 = $ENV::WORKER_IP
  - path: /etc/kubernetes/ssl/generate-tls-keys.sh
    owner: root
    permissions: 0755
    content: |
      #! /usr/bin/bash
      # Generates a set of TLS keys for this node to access the API server.
      set -e
      if [ ! -f /etc/kubernetes/ssl/worker.pem ]; then
        echo "Generating TLS keys."
        cd /etc/kubernetes/ssl
        openssl genrsa -out worker-key.pem 2048
        WORKER_IP=${1} openssl req -new -key worker-key.pem -out worker.csr -subj "/CN=worker" -config worker-openssl.cnf
        WORKER_IP=${1} openssl x509 -req -in worker.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out worker.pem -days 365 -extensions v3_req -extfile worker-openssl.cnf
      fi
      # Set permissions.
      sudo chmod 600 /etc/kubernetes/ssl/worker-key.pem
      sudo chown root:root /etc/kubernetes/ssl/worker-key.pem

  - path: /etc/kubernetes/ssl/ca.pem
    owner: core
    permissions: 0644
    content: |
      -----BEGIN CERTIFICATE-----
      -----END CERTIFICATE-----
  - path: /etc/kubernetes/ssl/ca-key.pem
    owner: core
    permissions: 0644
    content: |
      -----BEGIN RSA PRIVATE KEY-----
      -----END RSA PRIVATE KEY-----
```

Also we need to have scripts to check ports and install kubernetes.

```yaml
  - path: /opt/bin/wupiao
    permissions: '0755'
    content: |
      #!/bin/bash
      # [w]ait [u]ntil [p]ort [i]s [a]ctually [o]pen
      [ -n "$1" ] && \
        until curl -o /dev/null -sIf http://${1}; do \
          sleep 1 && echo .;
        done;
      exit $?

  - path: /opt/bin/kubernetes-install.sh
    owner: root
    permissions: 0755
    content: |
      #! /usr/bin/bash
      set -e

      if [ ! -f /opt/bin/kubelet ]; then
        echo "Kubenetes not installed - installing."

        export K8S_VERSION=$(curl -sS https://storage.googleapis.com/kubernetes-release/release/stable.txt)

        # Extract the Kubernetes binaries.
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubelet
        sudo wget -N -P /opt/bin http://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-proxy
        sudo chmod +x /opt/bin/kubelet /opt/bin/kube-proxy

        # Create required folders
        sudo mkdir -p /etc/kubernetes/manifests/
        sudo mkdir -p /etc/kubernetes/ssl/
      fi
```

Also we are going to run etcd on all nodes, but not the same way as on masters, here we will run it in proxy mode
# #############################
# TODO: write about proxy mode
# #############################

```yaml
  etcd2:
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    initial-cluster: etcd01=http://10.10.30.11:2380,etcd02=http://10.10.30.12:2380,etcd03=http://10.10.30.13:2380
    advertise-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    proxy: on
```


All SystemD units for minions is self describing and has nothing special.
The only thing worth mentionioing is DNS. It took me few days to understand this
chicken-egg problem. Kubernetes by itself has no DNS service, and you should install it afterwads.
In fact it is really easy - just create service and replication controller from receipts,
you can find in the kubernetes repository.
But, you need to decide about your internal domain zone and service IP address of your dns server
before you create your minions and hardcode it in your kubelet config.
So you need minion to deploy dns service, but you need to configure minion to know IP address of the future DNS.

```
    ExecStart=/opt/bin/kubelet \
    ...
    --cluster_dns=10.100.0.10 \
    --cluster_domain=cluster.local \
    ...

```

Complete cloud-config for minions could be found [on github](#)


# Checking that evething works
First of all we need to install `kubectl` tool to be able to talk to our cluster.

There are two ways you can talk to your cluster - first is secured channel using ssl keys we generated
earlie and the second one using insecure port 8080, but only in case you configured kube-apiserver
to listen anything other than localhost.

This is perfectly described at [documentation](#)
After following documentaion you should be able to execute `kubectl` command and have config at `~/.kube/config` with something like this:

```
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: ssl/ca.pem
    server: https://10.10.30.12:443
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    namespace: default
    user: admin
  name: dev
users:
- name: admin
  user:
    client-certificate: ssl/admin.pem
    client-key: ssl/admin-key.pem
current-context: dev
```

Now, if you run `kubectl get nodes` you should get back a list of your nodes with IP addresses.
If not - something went wrong and you should check the logs and try to fix it :)

Of course feel free to ping me and I'll try to help.

Since it already a huge post, I'm going to finish it here.
At this point we have fully operational cluster with several masters and several minions.
In the next post I will walk through deploying some basic services like: dns, heapster, dashboards and logs collecting.


## Problems during deploy
### Resetting endpoints for master service "kubernetes"
After deploing multiple masters I saw this message every minute or so

```bash
Apr 17 11:48:34 etcd01 kube-apiserver[1040]: W0417 11:48:34.571681    1040 controller.go:297] Resetting endpoints for master service "kubernetes" to &{{ } {kubernetes  default  f76faadd-0487-11e6-8614-000c29d236c7 7314 0 2016-04-17 10:34:29 +0000 UTC <nil> <nil> map[] map[]} [{[{10.10.30.11 <nil>}] [] [{https 443 TCP}]}]}
```

As said in https://github.com/openshift/origin/issues/6073 this happens when

> The controller running on each master that ensures that all masters IP's are present in the kubernetes endpoints looks at two things:

> 1. the current master's IP in the list

> 2. the number of IPs in the list exactly matching the masterCount

In my case it happened because I didn't add flag `--apiserver-count=N` to `kube-apiserver`

```
/opt/bin/kube-apiserver --apiserver-count=N
```

### Ignoring not a miss
This one I still didn't fix, luckily it doesn't cause any major problems.

```
Apr 18 07:59:11 etcd01 sdnotify-proxy[866]: I0418 07:59:11.903461 00001 vxlan.go:340] Ignoring not a miss: aa:25:34:2a:9c:7a, 10.3.77.10
```


One hint, which is kind of obvious, but in fact it saved me a lot of time.
Since you will create/destroy CoreOS instances tens or even hundreds of times - add this block to your ssh config to avoid deleting hosts from known_hosts file.

```
 Host 10.10.30.*
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null
```

