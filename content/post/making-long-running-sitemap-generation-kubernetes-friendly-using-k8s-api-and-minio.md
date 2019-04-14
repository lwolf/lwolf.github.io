+++
author = "Sergey Nuzhdin"
categories = ["docker", "kubernetes", "k8s-api", "python", "minio", "code"]
date = "2017-09-03"
description = ""
featured = "make-app-cloud-native.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Making long running sitemap generation kubernetes-friendly using k8s-api and Minio"

+++

Generation of sitemaps is usually not a problem, at least not until you have several millions of items in the database to iterate through. Currently, I have slightly more than 4M documents, so it takes time to regenerate. Previously all my sitemaps were stored on the disk and were served using simple [Nginx](https://nginx.org) container.
It worked like this:

- Sitemap-related containers have nodeSelector to assign to particular node with hostPath volume
- sitemaps are served from /data/sitemaps folder
- periodic task generates new sitemaps to /data/sitemaps-temp
- on completion, folders is atomically switched

It was fine in my previous VM-based cluster, which was on a single physical machine. After migration to the new multi-node cluster, I did not want to bind it to the single node to avoid downtime during node upgrade or maintenance. Since I do not have distributed block storage, I decided to serve it from the Minio which I already setup in a distributed fashion.

I thought that it is also a great opportunity to play with Kubernetes API in Python.

So the workflow became like this:

- Always have ingress pointing to sitemaps bucket in Minio
- Generate sitemaps inside the container
- Create bucket named sitemaps-<todays-date>
- Upload all of the sitemaps to the new bucket
- Make bucket public
- Switch ingress programmatically to point to the new bucket
- Delete old bucket


## Ingress

Ingress part is pretty usual, except for a few changes that I made to the metadata section.
First one is the `ingress.kubernetes.io/rewrite-target` rule.
What it does is replace URL part specified in the rule. For example in my case, the request like https://librer.io/sitemaps/sitemap-books.xml will be sent to Minio service as `/sitemaps-20170803/sitemap-books.xml`.
The second is the label with the bucket name put into labels, for easy filtering.


    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: libr-static
      labels:
        app: librerio
        role: static
        bucket: sitemaps-20170803
      annotations:
        ingress.kubernetes.io/rewrite-target: /sitemaps-20170803
        kubernetes.io/ingress.class: tectonic
        kubernetes.io/tls-acme: 'true'
    spec:
      tls:
      - hosts:
        - librer.io
        secretName: librerio-tls
      rules:
      - host: librer.io
        http:
          paths:
          - path: /sitemaps
            backend:
              serviceName: clustered-minio
              servicePort: 9000
    


## Kubernetes API in Python

Since most of the backend written in Python, python-client is the usual choice to access the API.

The first thing we need to do is access Kubernetes API. So we need to install the client.


    pip install kubernetes

Official client is hosted in the [kubernetes-incubator](https://github.com/kubernetes-incubator/client-python/) and has a lot of examples. Also, this [post](https://www.linux.com/learn/kubernetes/enjoy-kubernetes-python) could be useful if you’re trying k8s API for the first time.

Anyway, here are the basics.

To access the API we need to configure the client. For this, we need to import k8s config. There are two separate ways to access the cluster: using kubeconfig or serviceAccounts. First one is used mostly when you’re trying to access the cluster from the outside world, it is the same as `kubectl`.


    from kubernetes import config
    config.load_kube_config()

Another one, which you’ll be using more often, is using serviceAccounts from the pod running inside the cluster.


    from kubernetes import config
    config.load_incluster_config()

These are helper functions to make configuration easier. But, if you need, you can configure the client by setting option manually.

For testing, you can even run `kubectl proxy` and access the cluster like this


    >>> from kubernetes import client,config
    >>> client.Configuration().host="http://localhost:8001"
    

Using the client you can access all of the [API resources](https://github.com/kubernetes-incubator/client-python/tree/master/kubernetes#documentation-for-api-endpoints) available:


    # for example to get all nodes
    >>> v1=client.CoreV1Api()
    >>> v1.list_node()
    
    # or to get all namespace names
    >>> for ns in v1.list_namespace().items:
          print(ns.metadata.name)


## Back to the real-world

For this task we need to read and update ingress object. Make sure to give your serviceAccount corresponding roles/rolebindings in the RBAC-enabled cluster.
steps related to k8s API are:

- read the ingress
- extract current bucket name
- update ingress with the new bucket name

Ingresses are versioned as `extensions/v1beta1` so we need to create API instance with the corresponding version.


    kube_api = client.ExtensionsV1beta1Api()

Reading of the ingress by name and namespace could be done like this:


    ingress = kube_api.read_namespaced_ingress(ingress_name, namespace)

If you print this object you’ll see a big JSON, but all we need from it is just the name of the current bucket.


    old_bucket = ingress.metadata.labels.get('bucket')

Finally, we need to update the ingress with the information about new bucket name. For this, we’re going to use `patch_namespaced_ingress` method with a dictionary containing the fields we want to change.


    def update_bucket_in_ingress(bucket_name):
        body = {
            "metadata": {
                "labels": {
                    "bucket": bucket_name
                },
                "annotations": {
                    "ingress.kubernetes.io/rewrite-target": "/{}".format(bucket_name)
                }
            },
        }
        kube_api.patch_namespaced_ingress(ingress_name, namespace, body=body)


# tl;dr; Everything glued together

Here is the complete snippet of the code with Minio and k8s API related code.


    from kubernetes import config, client
    
    def create_bucket(minio_client, bucket_name):
        try:
            minio_client.make_bucket(
                bucket_name,
                location=Config.OBJECT_STORAGE_S3_BUCKET_LOCATION
            )
        except BucketAlreadyOwnedByYou as err:
            pass
        except BucketAlreadyExists as err:
            pass
        except ResponseError as err:
            raise
    
    def update_bucket_in_ingress(namespace, ingress_name, bucket_name):
        body = {
            "metadata": {
                "labels": {
                    "bucket": bucket_name
                },
                "annotations": {
                    "ingress.kubernetes.io/rewrite-target": "/{}".format(bucket_name)
                }
            },
        }
        kube_api.patch_namespaced_ingress(ingress_name, namespace, body=body)
    
    config.load_incluster_config()
    kube_api = client.ExtensionsV1beta1Api()
    namespace = os.environ['SITEMAPS_INGRESS_NAMESPACE']
    ingress_name = os.environ['SITEMAPS_INGRESS_NAME']
    
    ##### Steps not related to k8s api
    minio_client = Minio(host, access_key, secret_key)
    
    bucket_name = "sitemaps-{}".format(today)
    create_bucket(minio_client, bucket_name)
    
    tmp_store = '/tmp/sitemaps/{}'.format(today)
    os.makedirs(tmp_store, exist_ok=True)
    
    log.info("Going to generate sitemaps")
    for model, prefix in SITEMAPS:
        log.info("Generating sitemaps for %s..." % prefix)
        generate_items_sitemap(model, tmp_store, prefix)
        generate_sitemap_wrapper(tmp_store, prefix)
    
    log.info("Sitemaps generating completed. uploading to object storage...")
    for fl in glob.glob(os.path.join(tmp_store, '*')):
        minio_client.fput_object(
            bucket_name=bucket_name,
            object_name=fl.split('/')[-1],
            file_path=fl,
        )
    minio_client.set_bucket_policy(bucket_name, "*", Policy.READ_ONLY)
    ###
    
    ingress = kube_api.read_namespaced_ingress(ingress_name, namespace)
    old_bucket = ingress.metadata.labels.get('bucket')
    update_bucket_in_ingress(bucket_name)
    
    # Do not delete todays bucket on second run
    if old_bucket != bucket_name:
        try:
            minio_client.remove_bucket(old_bucket)
        except NoSuchBucket:
            log.info("Unable to delete old bucket, not found")
    

