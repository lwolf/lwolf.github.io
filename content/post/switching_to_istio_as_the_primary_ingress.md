+++
author = "Sergey Nuzhdin"
categories = ["kubernetes", "k8s", "service-mesh", "istio", "ingress"]
date = "2019-04-08"
description = "Switching to Istio as the primary ingress"
featured = "virtualservices-destrules.png"
featuredalt = "image source: https://istio.io/blog/2018/v1alpha3-routing/"
featuredpath = "date"
linktitle = ""
images = ["img/2019/04/virtualservices-destrules.png"]
draft = false
title = "Switching to Istio as the primary ingress"

+++

# Switching to Istio as the primary ingress

I’ve been following the news about [istio](https://istio.io/) since it’s first alpha release in 2017.  I think this project has a great future, because it solves a lot of pain points in the microservice based architecture, like auth, observability, fault-injection, etc.

But the fact is, I never actually tried it. Prior to v1.0 there were too many bugs and limitations to put it into production. So, when they finally released “production-ready” version I got quite exciting. It still took me a few months to find free time, but I finally gave it a try.

​​I decided to start my experimentations with Istio not from the usual bookstore example, but by making it the main ingress controller, responsible for all the traffic, incoming to my cluster. Previously, I used ingress-nginx, more precisely I had 2 ingress-controllers: one exposed to the Internet and one for internal-only services.  This post is about replacing both of them with Istio. 


## Install


> This post is referencing Istio version [1.1.0-rc.0](https://github.com/istio/istio/releases/tag/1.1.0-rc.0), unless explicitly stated otherwise

It is extremely easy and straight forward to install istio using helm chart, at least if you’re not trying to understand what all of the configuration options mean. I started by copying the default `values.yaml` file and enabling all the components (ingress, grafana, servicegraph, tracing, kiali, istiocoredns, istio_cni).


    # add istio helm repo
    $ helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.0-rc.0/charts
    
    # sync your index
    $ helm repo update
    
    # I usually save default values.yaml file to the values-custom.yaml to add to the repo
    $ helm upgrade --install -f istio/values-custom.yaml istio istio.io/istio --namespace istio-system

Assuming there were no problems during the installation, you now have a new namespace filled with containers.


    $ kubectl get pods -n istio-system
    grafana-d5d58cb7-fchjq                    1/1       Running     0          20h
    istio-citadel-c4489d577-wlwdh             1/1       Running     0          20h
    istio-egressgateway-5d4dd5f974-84btz      1/1       Running     0          20h
    istio-galley-57586fbc4-wgp55              1/1       Running     0          20h
    istio-ingress-6bf7fd96bd-v4s28            1/1       Running     0          20h
    istio-ingressgateway-6469b49cf-75pnb      1/1       Running     0          20h
    istio-pilot-5d76999bfc-lthr5              2/2       Running     0          20h
    istio-policy-5684c685cb-5qphq             2/2       Running     4          20h
    istio-sidecar-injector-58dff7458d-cqbhd   1/1       Running     0          20h
    istio-telemetry-7594984994-zgnlw          2/2       Running     4          20h
    istio-tracing-758ccfbfd9-9xdrc            1/1       Running     0          20h
    istiocoredns-c49bd87fb-75w8b              2/2       Running     0          20h
    kiali-986b769f4-cgxv4                     1/1       Running     0          20h
    prometheus-5c69f8d648-tm78k               1/1       Running     0          20h
    servicegraph-c69f96d97-r2qmh              1/1       Running     1          20h

and of course a lot of new services


    NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                      AGE
    grafana                  ClusterIP      10.233.47.239   <none>          3000/TCP                                                      1h
    istio-citadel            ClusterIP      10.233.52.66    <none>          8060/TCP,15014/TCP                                           1h
    istio-egressgateway      ClusterIP      10.233.39.186   <none>          80/TCP,443/TCP,15443/TCP                                     1h
    istio-galley             ClusterIP      10.233.6.227    <none>          443/TCP,15014/TCP,9901/TCP                                   1h
    istio-ingress            LoadBalancer   10.233.29.116   192.168.11.122   80:32000/TCP,443:32256/TCP                                   1h
    istio-ingressgateway     LoadBalancer   10.233.37.74    192.168.11.121   80:31380/TCP,443:31390/TCP                                   1h
    istio-pilot              ClusterIP      10.233.37.223   <none>          15010/TCP,15011/TCP,8080/TCP,15014/TCP                       1h
    istio-policy             ClusterIP      10.233.60.111   <none>          9091/TCP,15004/TCP,15014/TCP                                 1h
    istio-sidecar-injector   ClusterIP      10.233.47.222   <none>          443/TCP                                                      1h
    istio-telemetry          ClusterIP      10.233.3.12     <none>          9091/TCP,15004/TCP,15014/TCP,42422/TCP                       1h
    istiocoredns             ClusterIP      10.233.48.14    <none>          53/UDP,53/TCP                                                1h
    jaeger-agent             ClusterIP      None            <none>          5775/UDP,6831/UDP,6832/UDP                                   1h
    jaeger-collector         ClusterIP      10.233.2.234    <none>          14267/TCP,14268/TCP                                          1h
    jaeger-query             ClusterIP      10.233.46.47    <none>          16686/TCP                                                    1h
    kiali                    ClusterIP      10.233.31.94    <none>          20001/TCP                                                    1h
    prometheus               ClusterIP      10.233.51.54    <none>          9090/TCP                                                     1h
    servicegraph             ClusterIP      10.233.4.101    <none>          8088/TCP                                                     1h
    tracing                  ClusterIP      10.233.0.59     <none>          80/TCP                                                       1h
    zipkin                   ClusterIP      10.233.7.229    <none>          9411/TCP                                                     1h


> Services with the type LoadBalancer got their External IPs from the [metallb](https://github.com/google/metallb) load balancer set up in the cluster. 

some of the questions one may have at this point are:

- What’s the difference between **istio-ingress** and **istio-ingressgateway**, which one should you set in your DNS?
- How to access default istio services e.g kiali, grafana, prometheus?
- How to expose something to the outside world?


## Ingressing traffic: istio-ingress and istio-ingressgateway

The Ingress controller is, basically, a reverse-proxy that runs in a cluster and configures routing rules according to Ingress resources. 
Istio provides two ways of ingressing traffic into your cluster.  

First one, **istio-ingress**, is a traditional ingress controller like [nginx-ingress](https://github.com/kubernetes/ingress-nginx), [traefik](https://github.com/containous/traefik) or [controur](https://github.com/heptio/contour). 
This controller runs in your cluster and listens to all the changes to Ingress resources from the Kubernetes API and sends incoming traffic according to these rules. 
You can easily swap one ingress controller to another by just changing the annotation on your ingress objects:


    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      annotations:
        kubernetes.io/ingress.class: nginx

Istio documentation discourages use of this method as a “legacy way” and suggests using the second one.

The second one, **istio-ingressgateway,** is also an ingress controller, but unlike traditional ones, it does not rely on native Kubernetes Ingress objects. It uses its own custom resources: Gateway, VirtualService, DestinationRule, etc. 
Using custom resources provides more flexibility in configuring network policies, but also makes you write a few more YAMLs for each service. 
In this post, I’m going to use the second one.

## Exposing services using Istio-ingressgateway

To expose a service using ingressgateway you have to create at least 2 objects - Gateway and a VirtualService.
As an example, let’s expose the internal istio services.
Assuming that we have domain **example.com** we need to create a following gateway:


    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: service-gw
      namespace: default
    spec:
      selector:
        istio: ingressgateway
      servers:
        - port:
            number: 80
            name: http-system
            protocol: HTTP
          hosts:
          - "grafana.example.com"
          - "kiali.example.com"
          - "prometheus.example.com"
          - "jaeger.example.com"
    

Here we create a rule for the ingressgateway with the selector **istio: ingressgateway** to accept HTTP traffic on port 80 if SNI host matches the host in the list.

Few things to keep in mind:

- the selector is used by istio to select the ingressgateway. Keep this in mind if you have multiple ingressgateways
- Better to [avoid](https://github.com/istio/istio/issues/11509) using ***.domain** in hosts, unless you have only a single gateway.
- port naming is important, it is used in some routing logic [inside the istio](https://istio.io/help/faq/traffic-management/). 


  > The port names must be of the form `protocol`-`suffix` with `grpc`, `http`, `http2`, `https`, `mongo`, `redis`, `tcp`, `tls` or `udp` as the `protocol` in order to take advantage of Istio’s routing features.
  > 
  > For example, `name: http2-foo` or `name: http` are valid port names, but `name: http2foo` is not. If the port name does not begin with a recognized prefix or if the port is unnamed, traffic on the port will be treated as plain TCP traffic (unless the port explicitly uses Protocol: UDP to signify a UDP port).


Next, we need to create a VirtualService for each of the services we want to expose.
Here is a one for the Grafana:


    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: grafana
    spec:
      hosts:
        - grafana.example.com
      gateways:
        - service-gw.default
      http:
        - match:
            - uri:
                prefix: /
          route:
            - destination:
                port:
                  number: 3000
                host: grafana

Things to keep in mind here:

- If VirtualService and Gateway are located in the different namespaces, make sure to set gateway in the format of **gateway-name.namespace,** otherwise, you’ll be getting 404. 


## Istio and HTTPS

Exposing services to the world is cool, basics are working. But in 2019 everything should be exposed using HTTPS.
With Istio exposing site using HTTPS is in fact much more complicated and limited comparably with other ingresses. 
Let’s start with exposing our service using HTTPS and then I’ll list all the issues and limitations I faced with possible workarounds.

To update our gateway to listen to HTTPS traffic we need to have a valid certificate. I’m a big fan of [LetsEncrypt](https://letsencrypt.org/) and [cert-manager](https://github.com/jetstack/cert-manager), so I’m going to use it here.
You can install cert-manager as part of istio chart or you can use a dedicated chart for that. I already have cert-manager installed, so I will just add another certificate to the cluster.


    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Certificate
    metadata:
      name: example-com
    spec:
      secretName: example-com-certs
      issuerRef:
        name: letsencrypt-live
        kind: ClusterIssuer
      commonName: '*.example.com'
      dnsNames:
      - example.com
      acme:
        config:
        - dns01:
            provider: cloudflare-dns01
          domains:
          - '*.example.com'
          - example.com
    

Next step is to add this certificate to the igressgateway deployment. 


    istio-ingressgateway:
      secretVolumes:
        ...
        - name: example-com-certs
          secretName: example-com-certs
          mountPath: /etc/istio/example-com-certs
        ...

    
Now, we can update our gateway configuration to use HTTPS:


    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: service-gw
      namespace: istio-system
    spec:
      selector:
        istio: ingressgateway
      servers:
        - port:
            number: 443
            name: https-system
            protocol: HTTPS
          hosts:
          - "grafana.example.com"
          - "kiali.example.com"
          - "prometheus.example.com"
          - "jaeger.example.com"
          tls:
            mode: SIMPLE
            privateKey: /etc/istio/example-com-certs/tls.key
            serverCertificate: /etc/istio/example-com-certs/tls.crt
          

If everything went well, we can now access [Grafana](https://grafana.com/) using HTTPS with a valid certificate from the [LetsEncrypt](https://letsencrypt.org/).


## Issues

Now, let’s talk about the issues I have encountered during that short period of time I spent working with istio.

* [FIXED in 1.1] Manual update of the certificates. This is probably the most annoying thing. Having to update the deployment for each certificate, and restart ingressgateway to pick up it is not cool. This was finally fixed in 1.1 release. It is now possible to tell istio to use Kubernetes secret. [Detailed how-to is here](https://istio.io/docs/tasks/traffic-management/secure-ingress/sds/).

https://github.com/istio/istio/issues/9030

https://github.com/istio/istio/pull/11496

* [FIXED in 1.1] Another issue popped up when I tried to use the same port 443 for both PASSTHROUGH mode and SSL-termination, which is a perfectly reasonable scenario.  


https://discuss.istio.io/t/tls-modes-passthrough-and-simple/852

https://github.com/istio/istio/issues/11509

* [FIXED in [Kiali v0.16.2](https://github.com/kiali/kiali/releases/tag/v0.16.2)] I spent some time trying to figure out the correct way of specifying Gateway in the VirtualService. At some point, I thought that you can’t have Gateway and VirtualService in different namespaces. What made it even more confusing is [Kiali](https://www.kiali.io/). Whenever I tried to point VirtualService to the gateway in the different namespace I saw a validation error “virtualservice is pointing to non-existing gateway”
{{< figure src="/img/2019/04/kiali-error.png"  alt="kiali: virtualservice is pointing to non-existing gateway" >}}

This issue is fixed in the [Kiali v0.16.2](https://github.com/kiali/kiali/releases/tag/v0.16.2), which is not yet updated in istio chart.


## Conclusion

Istio is still a very young project. Reaching the 1.0 version was a huge step, but it still requires a lot of work. I only touched the ingress part of it and found that it lacks some of the basic functionality that one would expect from the Ingress controller.  
On the other hand, it is in a very active development phase. All the issues that I had in 1.0.* version was addressed and fixed in 1.1.0 release.
Another nice piece of software that comes with the Istio is Kiali, it looks really great and it should be very useful when I start using istio as a service mesh, not just an ingress.

