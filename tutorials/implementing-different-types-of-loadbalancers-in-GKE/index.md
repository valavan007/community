---
title: Implementing different types of load balancers in GKE
description: Deploying different kinds of load balancers like HTTPS-Internal and External LB, TCP- Intenral and Extenal LB using GKE Ingress 
author: valavan007
tags: GKE Ingress, Load Balancers, Internal Load Balancers
date_published: 2020-04-16
---

This guide provides a walk through of deploying a simple implementation on GKE and using the differnet types of Load Balancers which cater to differentuse cases. 

In GKE, Ingress controller is used to deploy Load Balancers on the GKE Platform. GKE ingress controllers currrently supports multiple load balancers like External HTTPS, Internal HTTPS, TCP/UDP External , TCP/UDP Internal and Container Native Load Balancer which are based on NEG. More details about the benefits of using Ccontainer Native Load Balancing and NEG can be found here[https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing].

This guide shows how to build a simple hello world application running in a single GKE cluster and exposing the service through the following load balancer option. Each type of load balancer will be implemented under a different namespace 

- Deploy an external HTTPS load balancer through standard GKE Ingress
- Deploy an external HTTPS load balancer using GKE ingress & Container Native Load balancing 
- Deploy an internal HTTPS load balancer using GKE ingress & Container Native Load balancing 
- Deploy an external TCP Load balancer 
- Deploy an internal TCP Load balancer 

### Prerequsites 

To perform the steps in the tutorials you will need a 

- A GKE cluster with version 1.15 and above 
- Clone the git repository(https://github.com/GoogleCloudPlatform/community)
- Navigate to ***cd tutorials/implemeting-diffferent-types-of-loadbalancers-in-GKE*** to use the K8S Manifest files 

### Preparing GCP environment 

1.  Before the next step, run `gcloud auth list` to ensure that your shell session is authenticated.

    If not you can use the `gcloud auth login` command to authenticate.
    
2.  In your shell, run the following commands to set the configuration context for `gcloud`:
```
	export PROJECT=$(gcloud info --format='value(config.project)')
	export ZONE=us-central1-f
        gcloud config set compute/zone [ZONE_NAME]
```
### Creating a Test GKE Cluster 

* Open Cloudshell to create a GKE Cluster 

	gcloud beta container clusters create hellow --release-channel=rapid --enable-ip-alias --network=default

Note: Internal HTTPS LB is currently in Beta and is supported on Rapid Release GKE versions 


