+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "kubernetes", "monitoring", "prometheus"]
date = "2017-07-21"
description = "Series of posts about migration from commercial monitoring systems to opensource. Replace NewRelic with Prometheus"
featured = "etcd-min-cover-min.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Going open-source in monitoring, part III: 10 most useful Grafana dashboards to monitor Kubernetes and services"
+++

There are dozens of ready dashboards available on [grafana.net/dashboards](https://grafana.net/dashboards) and Github.
Many of them just work, but many do not. In most cases, you just need to fix template variables, but some require deeper involvement. 
For example [dashboard](https://grafana.com/dashboards/455) for PostgreSQL monitoring. After import, it welcomes you with the error message “Datasource named ${DS_PROMETHEUS} was not found”. Setting the correct datasource name in dashboard settings does not help because it has an error in `__inputs`  declaration. The easiest way to fix it is to edit JSON before import.

It takes time to find useful dashboards, make them work for you and then choose one. For example, if you’ll search for Kubernetes dashboards with datasource Prometheus you’ll get, among others, 5 results with the same name.

{{< figure src="/img/2017/07/search-results-min.png"  alt="search results" >}}


In this post, I want to cover some of the most useful dashboards available to help you monitor your Kubernetes cluster and services deployed on it. 


# Cluster view

*Kubernetes cluster overview -* [#1621](https://grafana.com/dashboards/1621) or [#315](https://grafana.com/dashboards/315)

{{< figure src="/img/2017/07/cluster-metrics-min.png"  alt="Kubernetes cluster overview" >}}


This dashboard usually the first one you search for after deploying Kubernetes + Prometheus. Among the 5 shown in the screenshot above only 2 is really good. Actually, it is the same dashboard saved by two people. 


# Node view

Detailed node overview *-* [#1860](https://grafana.com/dashboards/1860)

{{< figure src="/img/2017/07/node-metrics-min.png"  alt="Node overview" >}}


 This one is great when you need to understand what is wrong with the particular node. It has much more stats compared to the cluster overview.

# Deployment metrics monitoring

*Kubernetes Deployment metrics -* [#741](https://grafana.com/dashboards/741)

{{< figure src="/img/2017/07/deploy-metrics-min.png"  alt="Kubernetes Deployment metrics" >}}


Next level of monitoring after the node - deployment. This dashboard will show you everything about selected deployment.

# Pod metrics monitoring

*Kubernetes Pod Metrics -* [#747](https://grafana.com/dashboards/747)

{{< figure src="/img/2017/07/pod-metrics-min.png"  alt="Kubernetes Pod Metrics" >}}


Pod level monitoring. Shows mostly the same stats as the deployment dashboard but on Pod level. `Pod info` row shows some random values in `Pod container` and `Pod IP Address` places, but other values seem fine.

# Application metrics monitoring

*Kubernetes App Metrics -* [#1471](https://grafana.com/dashboards/1471)

{{< figure src="/img/2017/07/pod-metrics-min.png"  alt="Kubernetes App Metrics" >}}


This one is good as an example of how to monitor your application deployed to Kubernetes. It has metrics like `request rate`, `error rate` and `response times` for your application alongside with different resource usage stats. 

# PostgreSQL

*Postgres Overview -* [*#455*](https://grafana.com/dashboards/455)

{{< figure src="/img/2017/07/postgresql-min.png"  alt="Postgres Overview" >}}


After you fix the broken variable declaration - it just works :)
I saved fixed one [here](https://github.com/lwolf/kube-monitoring/blob/master/dashboards/postgres-overview.json)

# ElasticSearch

Elasticsearch cluster overview [#266](https://grafana.com/dashboards/266)

{{< figure src="/img/2017/07/elasticsearch-min.png"  alt="Elasticsearch cluster overview" >}}


The best one so far is [266](https://grafana.com/dashboards/266). It supports most of the metrics needed and required least amount of tweaks to make it work.
There are several other options - [2322](https://grafana.com/dashboards/2322), [2347](https://grafana.com/dashboards/2347), [718](https://grafana.com/dashboards/718). But they either lack graphs or did not work at all for me.

# Redis

*Prometheus Redis -*  [#763](https://grafana.com/dashboards/763)

{{< figure src="/img/2017/07/redis-min.png"  alt="Prometheus Redis" >}}


# Memcached

Memcached node - [#37](https://grafana.com/dashboards/37)

{{< figure src="/img/2017/07/memcached-min.png"  alt="Memcached node" >}}

# Etcd
{{< figure src="/img/2017/07/etcd-min.png"  alt="Etcd" >}}


Monitoring of Etcd is covered on CoreOS website [here](https://coreos.com/etcd/docs/latest/op-guide/monitoring.html). Dashboard suggested in this manual has hardcoded datasource, so you have to edit it before import. So I made datasource configurable and saved it in the repo [here](https://github.com/lwolf/kube-monitoring/blob/master/dashboards/etcd-overview.json).
By default, Etcd is not monitored by Prometheus at all. So, you need to tell it where your Etcd lives.
If you have self-hosted Etcd annotations to the Etcd service will work, otherwise, you need to update prometheus config with the location on your Etcd nodes.

      - job_name: etcd
        static_configs:
        - targets: ['10.240.0.32:2379','10.240.0.33:2379','10.240.0.34:2379']


# Wrap Up

That’s it. At this point, we can monitor health of the Kubernetes cluster and several common services. 
Sources for PostgreSQL and Etcd dashboards, with changes I did, are available on [GitHub](https://github.com/lwolf/kube-monitoring/tree/master/dashboards).

Stay tuned.





