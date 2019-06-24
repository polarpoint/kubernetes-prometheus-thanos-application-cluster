---
id: 192
title: DevOps | Global Monitoring using Prometheus and Thanos
date: 2019-06-17T15:36:10+01:00
author: polarpoint_user
layout: post
guid: http://polarpoint.io/?p=192
permalink: /index.php/2019/06/17/devops-global-monitoring-using-prometheus-and-thanos/
image: /wp-content/uploads/2019/06/thanos-prometheus-grafana.png
categories:
  - Uncategorised
---
 

<p class="has-medium-font-size">
  <strong>Prometheus</strong>
</p>

Prometheus&nbsp;was originally conceived&nbsp; at Soundcloud Since its inception in 2012, many companies and organisations have adopted Prometheus.

Prometheus has become the standard tool for monitoring and alerting in the Cloud and container world’s.

Prometheus uses time series data model for metrics and events. Following are the key features of prometheus :-

  * a&nbsp;multi-dimensional&nbsp;data model (time series defined by metric name and set of key/value dimensions)
  * a&nbsp;flexible query language&nbsp;to leverage this dimensionality
  * no dependency on distributed storage;&nbsp;single server nodes are autonomous
  * time series collection happens via a&nbsp;pull model&nbsp;over HTTP
  * pushing time series&nbsp;is supported via an intermediary gateway
  * targets are discovered via&nbsp;service discovery&nbsp;or&nbsp;static configuration
  * multiple modes of&nbsp;graphing and dash boarding support
  * support for hierarchical and horizontal&nbsp;federation<figure class="wp-block-image">

<img src="http://polarpoint.io/wp-content/uploads/2019/06/image.png" alt="" class="wp-image-195" srcset="http://polarpoint.io/wp-content/uploads/2019/06/image.png 600w, http://polarpoint.io/wp-content/uploads/2019/06/image-300x203.png 300w" sizes="(max-width: 600px) 100vw, 600px" /> <figcaption>**Prometheus** **Architecture overview**</figcaption></figure> 

<p class="has-medium-font-size">
  <strong>Thanos</strong>
</p>

Thanos is a set of components that can be composed into a highly available metric system with unlimited storage capacity, which can be added seamlessly on top of existing Prometheus deployments.

Thanos leverages the Prometheus 2.0 storage format to cost-efficiently store historical metric data in any object storage while retaining fast query latencies. Additionally, it provides a global query view across all Prometheus installations and can merge data from Prometheus HA pairs on the fly.

Following are the key features of Thanos :-

  1. Global query view of metrics.
  2. Unlimited retention of metrics.
  3. High availability of components, including Prometheus.<figure class="wp-block-image">

<img src="http://polarpoint.io/wp-content/uploads/2019/06/image-1.png" alt="" class="wp-image-196" srcset="http://polarpoint.io/wp-content/uploads/2019/06/image-1.png 722w, http://polarpoint.io/wp-content/uploads/2019/06/image-1-300x225.png 300w" sizes="(max-width: 722px) 100vw, 722px" /> <figcaption>**Thanos Architecture overview**</figcaption></figure> 

<p class="has-medium-font-size">
  <strong>Thanos components&nbsp;</strong>
</p>

Thanos is made of a set of components with each filling a specific role.

  * **Sidecar**: connects to Prometheus and reads its data for query and/or upload it to cloud storage
  * **Store Gateway**: exposes the content of a cloud storage bucket
  * **Compactor**: compact and downsample data stored in remote storage
  * **Receiver**: receives data from Prometheus’ remote-write WAL, exposes it and/or upload it to cloud storage
  * **Ruler**: evaluates recording and alerting rules against data in Thanos for exposition and/or upload
  * **Query Gateway**: implements Prometheus’ v1 API to aggregate data from the underlying components

<p class="has-medium-font-size">
  <strong>Sidecar</strong>
</p>

Thanos integrates with existing Prometheus servers through a&nbsp;sidecar process which runs in the same pod as the Prometheus server.

The purpose of the Sidecar is to backup Prometheus data into an Object Storage bucket, and giving other Thanos components access to the Prometheus instance the Sidecar is attached to via a gRPC API.

<p class="has-medium-font-size">
  <strong>Application Kubernetes Clusters</strong>
</p>

To be able to get a global view of all the different environments ranging from dev through to prod we configure and install the Prometheus Operator, Prometheus components, Thanos Sidecar and ingress into each cluster.

**Prometheus Operator**

##### Configuring Thanos Object Storage

Thanos expects a Kubernetes Secret containing the Thanos configuration. Inside this secret you configure how to run Thanos with your object storage.

Once you have written your configuration save it to a file called `thanos-storage-config.yaml`

  
Here&#8217;s are a few examples for the major cloud providers:-

<pre class="wp-block-preformatted">type: s3
config:
  bucket: thanos
  endpoint: aws.polarpoint.io
  access_key: XXX
  secret_key: XXX</pre>

<pre class="wp-block-preformatted">type: GCS
config:
  bucket: ""
  service_account: ""</pre>

<pre class="wp-block-preformatted">type: AZURE
 config:
   storage_account: "XXX"
   storage_account_key: "XXX"
   container: "thanos"</pre>

<pre class="wp-block-preformatted">kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config.yaml --namespace default</pre>

As well as the Blob storage configuration we want to ensure all communication is secured using mTLS creating a tls secret signed with the same CA certificate as will be used for the ingress controller and another for the CA certificate.

<pre class="wp-block-preformatted">kubectl create secret tls -n default thanos-ingress-secret --key dev-client.key --cert dev-client.cert

kubectl create secret generic -n default thanos-ca-secret --from-file=ca.crt=cacerts.cer</pre>

Using the helm chart for Prometheus Operator and the following values file (prometheus-operator-thanos-values.yaml)

<pre class="wp-block-preformatted">prometheus:
   prometheusSpec:
     replicas: 2      
     retention: 12h   # we only need a few hours of retention, since the rest is uploaded to blob
     image:
       tag: v2.10.0    
     serviceMonitorNamespaceSelector:  # find target config from multiple namespaces
       any: true
     thanos:         # add Thanos Sidecar
       tag: v0.5.0   
       objectStorageConfig: # blob storage to upload metrics
         key: thanos.yaml
         name: thanos-storage-config
 grafana:          
   enabled: false</pre>

<pre class="wp-block-preformatted">helm install  --name dev-prom stable/prometheus-operator -f prometheus-operator-thanos-values.yaml  --tiller-namespace=default</pre>

We now have the Prometheus Operator installed in the application cluster.

<pre class="wp-block-preformatted">kubectl get svc -n default -o wide

dev-prom-kube-state-metrics                        ClusterIP      xxxx              8080/TCP                     29d   app=kube-state-metrics,release=int-prom
 dev-prom-prometheus-node-exporter                  ClusterIP      xxxx               9100/TCP                     29d   app=prometheus-node-exporter,release=dev-prom
 dev-prom-prometheus-operat-alertmanager            ClusterIP      xxxx             9093/TCP                     29d   alertmanager=dev-prom-prometheus-operat-alertmanager,app=alertmanager
 dev-prom-prometheus-operat-operator                ClusterIP      xxxx              8080/TCP                     29d   app=prometheus-operator-operator,release=dev-prom
 dev-prom-prometheus-operat-prometheus              ClusterIP      xxxx             9090/TCP                     29d   app=prometheus,prometheus=dev-prom-prometheus-operat-prometheus
 kubernetes                                         ClusterIP      xxxx                 443/TCP                      29d   
 prometheus-operated                                ClusterIP      None                     9090/TCP                     11d   app=prometheus
 thanos-sidecar-0                                   ClusterIP      xxxx              10901/TCP                    29d   statefulset.kubernetes.io/pod-name=prometheus-dev-prom-prometheus-operat-prometheus-0</pre>

To enable Thanos to be able to scrape the application cluster metrics we need to allow&nbsp;**Thanos Store Gateway**&nbsp;access to the &nbsp;**Thanos Sidecars** running in each application cluster, we need to expose them via an&nbsp;**Ingress** secured with MTLS using the tls and cacert secrets defined above.

<pre class="wp-block-preformatted">kubectl apply -f thanos-ingress-rules.yaml -n default</pre>

<pre class="wp-block-preformatted">kubectl get ingress</pre>

<pre class="wp-block-preformatted"><br /> NAME               HOSTS               ADDRESS   PORTS     AGE<br /> thanos-sidecar-0   prom.dev-polarpoint.local             80, 443   29d</pre>
